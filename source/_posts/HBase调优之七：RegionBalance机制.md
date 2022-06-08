---
title: HBase调优之七：RegionBalance机制及实践
date: 2022-06-06 16:27:35.107
updated: 2022-06-06 16:27:35.107
url: /archives/region_balance
categories: [HBase调优]
tags: [hbase,性能调优]
description: 在HBase中，我们存储的数据是时序数据。时序数据的特点是，只关心实时数据，历史数据一般不关心。因此，我们在设计RowKey时，充分根据时间特性来进行设置：bucket_time_xxx。
top: 10
---

## 背景

在HBase中，我们存储的数据是时序数据。时序数据的特点是，只关心实时数据，历史数据一般不关心。

因此，我们在设计RowKey时，充分根据时间特性来进行设置：bucket_time_xxx

其中，bucket个数是固定的，取值范围：[1, 512]。主要用来将数据均匀的分散在不同的Region。

但是随着时间的推移，Region发生split后，旧时间段的Region，不会再有写入。即：它只提供查询，不提供实时写入。而我们的业务场景是写多读少场景。

这就导致，我们的RegionServer的Region虽然分配很均匀，但是写入差距却很大。

![](../images/Dingtalk_20220530150011.jpg)

![](../images/Dingtalk_20220530150107.jpg)



## Balancer源码分析

### balancer触发时机

**一、手动触发**

在hbase shell中执行

```shell
hbase> balancer
```

hbase shell中提交的所有命令，都是执行在HMaster的main函数中。

```java
public static void main(String [] args) {
    LOG.info("STARTING service " + HMaster.class.getSimpleName());
    VersionInfo.logVersion();
    new HMasterCommandLine(HMaster.class).doMain(args);
}
```



**二、定时触发**

hbase提供了配置项：hbase.balancer.period

每隔5分钟，会自动检查一次是否需要进行rebalance。

```java
public class BalancerChore extends ScheduledChore {
    private static final Logger LOG = LoggerFactory.getLogger(BalancerChore.class);

    private final HMaster master;

    public BalancerChore(HMaster master) {
        // 每隔5分钟触发一次
        super(master.getServerName() + "-BalancerChore", master, master.getConfiguration().getInt(
            HConstants.HBASE_BALANCER_PERIOD, HConstants.DEFAULT_HBASE_BALANCER_PERIOD));
        this.master = master;
    }

    @Override
    protected void chore() {
        try {
            master.balance();
        } catch (IOException e) {
            LOG.error("Failed to balance.", e);
        }
    }
}
```





### 执行过程

触发balance后，实际执行入口是HMaster的balance方法。有无参和是否强制执行。默认不强制执行。强制执行是说，当前如果存在RIT(正在发生split)时，也能继续执行。

#### **初始化**

HMaster启动时，会调用initializeZKBasedSystemTrackers()方法初始化balancer。

```java
protected void initializeZKBasedSystemTrackers()
    throws IOException, InterruptedException, KeeperException, ReplicationException {
 	// balancer相关代码，省略了其它逻辑
    //初始化balancer
    this.balancer = LoadBalancerFactory.getLoadBalancer(conf);
    //Factory中，使用的default是StochasticLoadBalancer
}
```

这里面的balancer是：LoadBalancer 类型。目前，HBase提供了4种balance方式。依赖关系如下：

![](../images/hbase/balance类图.jpg)

- RSGroupBasedLoadBalancer

  > Region分组时可以使用它

- StochasticLoadBalancer

  > 默认balance策略，HBase中最好用的

- FavoredStochasticBalancer

  > 

- SimpleLoadBalancer

  > 最简单的平衡策略，单纯的通过RegionCount来平衡

- FavoredNodeLoadBalancer

  > 为每个Region分配首选RS。



HMaster启动后，会调用finishActiveMasterInitialization()方法做一些初始化动作。其中，balance的初始化也在其中。

```java
private void finishActiveMasterInitialization(MonitoredTask status)
      throws IOException, InterruptedException, KeeperException, ReplicationException {
 	// balancer相关代码，省略了其它逻辑
    //初始化balance
    this.balancer.setMasterServices(this);
    this.balancer.setClusterMetrics(getClusterMetricsWithoutCoprocessor());
    this.balancer.initialize();
}
```



#### **检查是否需要执行balance**

当触发balancer后，会检查是否需要进行。

```java
public boolean balance(boolean force) throws IOException {
    int maxRegionsInTransition = getMaxRegionsInTransition();
    synchronized (this.balancer) {
        // 关注点1：未开启balancer，不进行balancer
        if (!this.loadBalancerTracker.isBalancerOn()) return false;
        
        // 关注点2：存在meta表的RIT 或 其它表的RIT但未force，不进行balancer
        if (this.assignmentManager.hasRegionsInTransition()) {	//存在RIT时
            List<RegionStateNode> regionsInTransition = assignmentManager.getRegionsInTransition();
            
            // 如果存在meta表进行RIT，则即使是force，也返回失败
            boolean metaInTransition = assignmentManager.isMetaRegionInTransition();
            String prefix = force && !metaInTransition ? "R" : "Not r";
            LOG.info(prefix + "unning balancer because " + regionsInTransition.size() +
                     " region(s) in transition: " + toPrint + (truncated? "(truncated list)": ""));
            if (!force || metaInTransition) return false;
        }
        
        // 关注点3：存在异常的RS，不进行balancer
        if (this.serverManager.areDeadServersInProgress()) {
            LOG.info("Not running balancer because processing dead regionserver(s): " +
                     this.serverManager.getDeadServers());
            return false;
        }
        
        // 关注点4：存在Region是Open状态，但分配到该Region的RS不存在（一般是异常情况，造成不一致的代码太复杂，没多做关注。假使出现了这种情况，将该Region disable后，再open，即能分配到合理的RS上。）
        Map<ServerName, ServerMetrics> onlineServers = serverManager.getOnlineServers();
        int regionNotOnOnlineServer = 0;
        for (RegionState regionState : assignmentManager.getRegionStates().getRegionStates()) {
            if (regionState.isOpened() && !onlineServers
                .containsKey(regionState.getServerName())) {
                LOG.warn("{} 's server is not in the online server list.", regionState);
                regionNotOnOnlineServer++;
            }
        }
        if (regionNotOnOnlineServer > 0) {
            LOG.info("Not running balancer because {} regions found not on an online server",
                     regionNotOnOnlineServer);
            return false;
        }

        
        // 关注点5：生成Balancer计划，支持按表去进行balance
        boolean isByTable = getConfiguration().getBoolean("hbase.master.loadbalance.bytable", false);
        Map<TableName, Map<ServerName, List<RegionInfo>>> assignments =
            this.assignmentManager.getRegionStates().getAssignmentsForBalancer(isByTable);
        for (Map<ServerName, List<RegionInfo>> serverMap : assignments.values()) {
            serverMap.keySet().removeAll(this.serverManager.getDrainingServersList());
        }

        //Give the balancer the current cluster state.
        this.balancer.setClusterMetrics(getClusterMetricsWithoutCoprocessor());
        this.balancer.setClusterLoad(assignments);

        List<RegionPlan> plans = new ArrayList<>();
        for (Entry<TableName, Map<ServerName, List<RegionInfo>>> e : assignments.entrySet()) {
           	// 关注点6：是否需要进行balancer在此处判断，如果需要balance，则生成RegionPlan
            List<RegionPlan> partialPlans = this.balancer.balanceCluster(e.getKey(), e.getValue());
            if (partialPlans != null) {
                plans.addAll(partialPlans);
            }
        }

        // 开始进行balance
        long balanceStartTime = System.currentTimeMillis();
        long cutoffTime = balanceStartTime + this.maxBlancingTime;
        int rpCount = 0;  // number of RegionPlans balanced so far
        if (plans != null && !plans.isEmpty()) {
            int balanceInterval = this.maxBlancingTime / plans.size();
            LOG.info("Balancer plans size is " + plans.size() + ", the balance interval is "
                     + balanceInterval + " ms, and the max number regions in transition is "
                     + maxRegionsInTransition);

            for (RegionPlan plan: plans) {
                LOG.info("balance " + plan);
                //TODO: bulk assign
                try {
                    //关注点7：实际去进行balance，把当前RS上的Region移动到别的RS上。
                    this.assignmentManager.moveAsync(plan);
                } catch (HBaseIOException hioe) {
                    //should ignore failed plans here, avoiding the whole balance plans be aborted
                    //later calls of balance() can fetch up the failed and skipped plans
                    LOG.warn("Failed balance plan: {}, just skip it", plan, hioe);
                }
                //rpCount records balance plans processed, does not care if a plan succeeds
                rpCount++;

                balanceThrottling(balanceStartTime + rpCount * balanceInterval, maxRegionsInTransition,
                                  cutoffTime);

                // if performing next balance exceeds cutoff time, exit the loop
                if (rpCount < plans.size() && System.currentTimeMillis() > cutoffTime) {
                    // TODO: After balance, there should not be a cutoff time (keeping it as
                    // a security net for now)
                    LOG.debug("No more balancing till next balance run; maxBalanceTime="
                              + this.maxBlancingTime);
                    break;
                }
            }
        }
    }
    // If LoadBalancer did not generate any plans, it means the cluster is already balanced.
    // Return true indicating a success.
    return true;
}
```



关注点6中的代码实现细节：默认使用的是：StochasticLoadBalancer，看它里面生成balancer计划逻辑。

```java
public synchronized List<RegionPlan> balanceCluster(Map<ServerName,
                                                    List<RegionInfo>> clusterState) {
    // 关注点1：先对master上的 region 进行平衡。具体平衡逻辑参考下面。
    List<RegionPlan> plans = balanceMasterRegions(clusterState);
    if (plans != null || clusterState == null || clusterState.size() <= 1) {
        // 如果master上需要进行balance，则这次balance只在master节点上进行，其它RegionServer上不进行balance。
        // 我们当前默认是不需要在master节点上存放region，因此，这步会跳过。
        return plans;
    }

    if (masterServerName != null && clusterState.containsKey(masterServerName)) {
        if (clusterState.size() <= 2) {
            // 除了master之外，如果只剩下一个RS，也不进行balance。
            return null;
        }
        clusterState = new HashMap<>(clusterState);
        clusterState.remove(masterServerName);
    }

    // On clusters with lots of HFileLinks or lots of reference files,
    // instantiating the storefile infos can be quite expensive.
    // Allow turning this feature off if the locality cost is not going to
    // be used in any computations.
    RegionLocationFinder finder = null;
    if ((this.localityCost != null && this.localityCost.getMultiplier() > 0)
        || (this.rackLocalityCost != null && this.rackLocalityCost.getMultiplier() > 0)) {
        finder = this.regionFinder;
    }

    //The clusterState that is given to this method contains the state
    //of all the regions in the table(s) (that's true today)
    // Keep track of servers to iterate through them.
    Cluster cluster = new Cluster(clusterState, loads, finder, rackManager);

    long startTime = EnvironmentEdgeManager.currentTime();

    initCosts(cluster);

    // 关注点2：判断是否需要balance。核心代码如下。
    if (!needsBalance(cluster)) {
        return null;
    }

    double currentCost = computeCost(cluster, Double.MAX_VALUE);
    curOverallCost = currentCost;
    for (int i = 0; i < this.curFunctionCosts.length; i++) {
        curFunctionCosts[i] = tempFunctionCosts[i];
    }
    double initCost = currentCost;
    double newCost = currentCost;

    // 计划需要多少步完成本次balance
    long computedMaxSteps;
    if (runMaxSteps) {
        computedMaxSteps = Math.max(this.maxSteps,
                                    ((long)cluster.numRegions * (long)this.stepsPerRegion * (long)cluster.numServers));
    } else {
        long calculatedMaxSteps = (long)cluster.numRegions * (long)this.stepsPerRegion *
            (long)cluster.numServers;
        computedMaxSteps = Math.min(this.maxSteps, calculatedMaxSteps);
        if (calculatedMaxSteps > maxSteps) {
            LOG.warn("calculatedMaxSteps:{} for loadbalancer's stochastic walk is larger than "
                     + "maxSteps:{}. Hence load balancing may not work well. Setting parameter "
                     + "\"hbase.master.balancer.stochastic.runMaxSteps\" to true can overcome this issue."
                     + "(This config change does not require service restart)", calculatedMaxSteps,
                     maxRunningTime);
        }
    }
    LOG.info("start StochasticLoadBalancer.balancer, initCost=" + currentCost + ", functionCost="
             + functionCost() + " computedMaxSteps: " + computedMaxSteps);

    // Perform a stochastic walk to see if we can get a good fit.
    long step;

    // 关注点3：生成具体的执行动作，并进行模拟执行
    for (step = 0; step < computedMaxSteps; step++) {
        // 生成执行动作，核心代码在下面
        Cluster.Action action = nextAction(cluster);

        if (action.type == Type.NULL) {
            continue;
        }

        // 模拟执行
        cluster.doAction(action);
        // 模拟执行后更新收益情况
        updateCostsWithAction(cluster, action);

        newCost = computeCost(cluster, currentCost);

        // Should this be kept?
        // 模拟执行后是否有收益，如果有，则往下执行，否则回滚收益：即本次action不进行
        if (newCost < currentCost) {
            currentCost = newCost;

            // save for JMX
            curOverallCost = currentCost;
            for (int i = 0; i < this.curFunctionCosts.length; i++) {
                curFunctionCosts[i] = tempFunctionCosts[i];
            }
        } else {
            // Put things back the way they were before.
            // TODO: undo by remembering old values
            Action undoAction = action.undoAction();
            cluster.doAction(undoAction);
            updateCostsWithAction(cluster, undoAction);
        }

        if (EnvironmentEdgeManager.currentTime() - startTime >
            maxRunningTime) {
            break;
        }
    }
    long endTime = EnvironmentEdgeManager.currentTime();

    metricsBalancer.balanceCluster(endTime - startTime);

    // update costs metrics
    updateStochasticCosts(tableName, curOverallCost, curFunctionCosts);
    if (initCost > currentCost) {
        plans = createRegionPlans(cluster);
        LOG.info("Finished computing new load balance plan. Computation took {}" +
                 " to try {} different iterations.  Found a solution that moves " +
                 "{} regions; Going from a computed cost of {}" +
                 " to a new cost of {}", java.time.Duration.ofMillis(endTime - startTime),
                 step, plans.size(), initCost, currentCost);
        return plans;
    }
    LOG.info("Could not find a better load balance plan.  Tried {} different configurations in " +
             "{}, and did not find anything with a computed cost less than {}", step,
             java.time.Duration.ofMillis(endTime - startTime), initCost);
    return null;
}
```



**master节点上的balancer操作**

```java
// 关注点1代码实现细节
protected List<RegionPlan> balanceMasterRegions(Map<ServerName, List<RegionInfo>> clusterMap) {
    if (masterServerName == null || clusterMap == null || clusterMap.size() <= 1) return null;
    List<RegionPlan> plans = null;
    List<RegionInfo> regions = clusterMap.get(masterServerName);
    if (regions != null) {
        Iterator<ServerName> keyIt = null;
        for (RegionInfo region: regions) {
            // 核心代码：某个region是否有必要存在master上。
            /**
             * 判断逻辑：检查配置，是否开启系统表的region放在master。如果开启，并且是系统表，则需要放在master。默认关闭。
             * hbase.balancer.tablesOnMaster.systemTablesOnly
             **/ 
            if (shouldBeOnMaster(region)) continue;

            // Find a non-master regionserver to host the region
            if (keyIt == null || !keyIt.hasNext()) {
                keyIt = clusterMap.keySet().iterator();
            }
            ServerName dest = keyIt.next();
            if (masterServerName.equals(dest)) {
                if (!keyIt.hasNext()) {
                    keyIt = clusterMap.keySet().iterator();
                }
                dest = keyIt.next();
            }

            // Move this region away from the master regionserver
            RegionPlan plan = new RegionPlan(region, masterServerName, dest);
            if (plans == null) {
                plans = new ArrayList<>();
            }
            plans.add(plan);
        }
    }
    for (Map.Entry<ServerName, List<RegionInfo>> server: clusterMap.entrySet()) {
        if (masterServerName.equals(server.getKey())) continue;
        for (RegionInfo region: server.getValue()) {
            if (!shouldBeOnMaster(region)) continue;

            // Move this region to the master regionserver
            RegionPlan plan = new RegionPlan(region, server.getKey(), masterServerName);
            if (plans == null) {
                plans = new ArrayList<>();
            }
            plans.add(plan);
        }
    }
    return plans;
}
```



**needsBalance逻辑**

```java
// 1. 初始化Cost
public void init(){
    costFunctions = new CostFunction[]{
        new RegionCountSkewCostFunction(conf),			// Region数目维度
        new PrimaryRegionCountSkewCostFunction(conf),	// 
        new MoveCostFunction(conf),						// 迁移Region维度
        localityCost,									// 文件本地化维度
        rackLocalityCost,
        new TableSkewCostFunction(conf),				// 表维度
        regionReplicaHostCostFunction,					// 副本
        regionReplicaRackCostFunction,					
        regionLoadFunctions[0],							// 负载维度，读写等
        regionLoadFunctions[1],
        regionLoadFunctions[2],
        regionLoadFunctions[3],
    };
}

// 2. 是否需要进行balance
protected boolean needsBalance(Cluster cluster) {
    ClusterLoadState cs = new ClusterLoadState(cluster.clusterState);
    if (cs.getNumServers() < MIN_SERVER_BALANCE) {
        // RS个数小于2，我们已经大于2了。
        if (LOG.isDebugEnabled()) {
            LOG.debug("Not running balancer because only " + cs.getNumServers()
                      + " active regionserver(s)");
        }
        return false;
    }
    if (areSomeRegionReplicasColocated(cluster)) {
        return true;
    }

    // 核心代码
    double total = 0.0;
    float sumMultiplier = 0.0f;
    for (CostFunction c : costFunctions) {
        float multiplier = c.getMultiplier();
        if (multiplier <= 0) {
            continue;
        }
        if (!c.isNeeded()) {
            LOG.debug("{} not needed", c.getClass().getSimpleName());
            continue;
        }
        sumMultiplier += multiplier;
        total += c.cost() * multiplier;
    }

    // minCostNeedBalance默认值大小为：0.05，可通过hbase.master.balancer.stochastic.minCostNeedBalance配置。
    // 官网https://issues.apache.org/jira/browse/HBASE-22349
    if (total <= 0 || sumMultiplier <= 0
        || (sumMultiplier > 0 && (total / sumMultiplier) < minCostNeedBalance)) {
        if (LOG.isTraceEnabled()) {
            final String loadBalanceTarget =
                isByTable ? String.format("table (%s)", tableName) : "cluster";
            LOG.trace("Skipping load balancing because the {} is balanced. Total cost: {}, "
                      + "Sum multiplier: {}, Minimum cost needed for balance: {}", loadBalanceTarget, total,
                      sumMultiplier, minCostNeedBalance);
        }
        return false;
    }
    return true;
}
```



**几个重要的CostFunction**

![](../images/hbase/a3d7392a4e96ced252a0f425ed1ebb4a40e519ad.png)

CostFunction是一个判断代价的函数，取值范围是0到1，值越小，说明越平衡。

对于MoveCostFunction来说，初始状态就是最佳状态，因为不需要任何移动，而移动所有region则是最差状态。
而对于其它CostFunciton，则需要分别去考虑最差和最佳时的值具体是什么，比如对于RegionCountSkewCostFunction，最差状态是所有reigon都集中在单个regionserver上，而最佳状态是所有regionserver上的region数量都等于平均数。

值得注意的是，对于ServerLocalityCostFunction及其它数据本地化相关因子来说，最佳状态并非是完全本地化，而是基于当前hdfs的block分布状态，所能达到的最大本地化值，

收益的判断需要对doAction之前和之后的totalCost进行比较，而totalCost是对各个因子的cost加权求和得到的，如下所示，实线框为初始状态时的cost，模拟执行了action之后得到了虚线框代表的cost，有的增加有的减小，最后综合下来的增减情况就代表了执行该action是否有价值。



![balancer_](../images/hbase/07d2b94044f25913dd473ed0abdc49f9fda67da0.png)

CandidateGenerator生成action的次数有一定限制，称为maxStep，该值与集群配置以及集群规模相关。



核心关注的几个Region：

RegionCountSkewCostFunction、MoveCostFunction、ServerLocalityCostFunction、RackLocalityCostFunction、TableSkewCostFunction、ReadRequestCostFunction、WriteRequestCostFunction、MemStoreSizeCostFunction、StoreFileCostFunction



**MoveCostFunction**

```java
double cost() {
    // 获取最大移动个数。[x * 0.25, 600] 
    int maxMoves = Math.max((int) (cluster.numRegions * maxMovesPercent),
                            DEFAULT_MAX_MOVES);

    double moveCost = cluster.numMovedRegions;

    if (moveCost > maxMoves) {
        return 1000000;   // return a number much greater than any of the other cost
    }

    return scale(0, Math.min(cluster.numRegions, maxMoves), moveCost);
}

protected double scale(double min, double max, double value) {
    if (max <= min || value <= min) {
        return 0;
    }
    if ((max - min) == 0) return 0;

    return Math.max(0d, Math.min(1d, (value - min) / (max - min)));
}

// numMovedRegions的计算方式。
public void doAction(){
    if (initialRegionIndexToServerIndex[region] == newServer) {
        numMovedRegions--; //region moved back to original location
    } else if (oldServer >= 0 && initialRegionIndexToServerIndex[region] == oldServer) {
        numMovedRegions++; //region moved from original location
    }
}
```



**RegionCountSkewCostFunction（权重很高）**

```java
double cost() {
    if (stats == null || stats.length != cluster.numServers) {
        // numServers会在每次Balance时创建
        stats = new double[cluster.numServers];
    }

    for (int i =0; i < cluster.numServers; i++) {
        stats[i] = cluster.regionsPerServer[i].length;
    }

    return costFromArray(stats);
}

// 以线上1990个Region和16个RS计算
protected double costFromArray(double[] stats) {
    double totalCost = 0;
    // 所有的region总数量
    
    double total = getSum(stats);

    // 所有的RS数
    double count = stats.length;
    
    // 平均每个RS个数 
    double mean = total/count;

    // 最差分布的cost，即有一个server上负载了所有region，其他server上region为0的cost
    double max = ((count - 1) * mean) + (total - mean);

    // It's possible that there aren't enough regions to go around
    // 最优分布的cost
    double min;
    if (count > total) {
        min = ((count - total) * mean) + ((1 - mean) * total);
    } else {
        // Some will have 1 more than everything else. 
        int numHigh = (int) (total - (Math.floor(mean) * count));
        int numLow = (int) (count - numHigh);
        min = (numHigh * (Math.ceil(mean) - mean)) + (numLow * (mean - Math.floor(mean)));

    }
    min = Math.max(0, min);
    
    // 计算当前的cost
    for (int i=0; i<stats.length; i++) {
        double n = stats[i];
        double diff = Math.abs(mean - n);
        totalCost += diff;
    }

    // 计算最终的cost
    double scaled =  scale(min, max, totalCost);
    return scaled;
}
```

hbase.master.balancer.stochastic.regionCountCost。regionCountCost权重，默认值为500；

用线下环境做测试：

![](../images/hbase/lQLPJxZc3z7s6bHM2c0CGLBFpW5qBdzkrQKZOlXxANYA_536_217.png)

total：2790  count：3  mean：930

max = (3-1) * 930 + 2790 - 930 = 3720

min = 1

当前cost：（1395 -930） * 2 + 930 = 1860

该CostFunction返回的cost：1860 / 3720 = 0.5

重启停掉的Region后：

![](../images/hbase/1654081128(1).jpg)



**TableSkewCostFunction**

单个表在某个RS上最大的Region数。

```java
double cost() {
    double max = cluster.numRegions;	// 总Region数
    double min = ((double) cluster.numRegions) / cluster.numServers;	//每个Region的平均数
    double value = 0;

    // 各table在单个RS中的最大region数
    for (int i = 0; i < cluster.numMaxRegionsPerTable.length; i++) {
        value += cluster.numMaxRegionsPerTable[i];
    }

    return scale(min, max, value);
}
```



**负载相关的CostFunction**

Read、Write、MemStore、StoreFile实现逻辑一致。

```java
double cost() {
    if (clusterStatus == null || loads == null) {
        return 0;
    }

    if (stats == null || stats.length != cluster.numServers) {
        stats = new double[cluster.numServers];
    }

    // 遍历所有的RS
    for (int i =0; i < stats.length; i++) {
        //Cost this server has from RegionLoad
        long cost = 0;

        // for every region on this server get the rl
        for(int regionIndex:cluster.regionsPerServer[i]) {
            // regionLoad，默认会取最近15次的流量统计做平均值。15次由StochasticLoadBalancer的numRegionLoadsToRemember参数配置。
            // hbase.master.balancer.stochastic.numRegionLoadsToRemember，这个RegionLoad的更新策略在下面分析。
            Collection<BalancerRegionLoad> regionLoadList =  cluster.regionLoads[regionIndex];

            // Now if we found a region load get the type of cost that was requested.
            if (regionLoadList != null) {
                // 计算每个Region的Cost。核心代码在下面。
                cost = (long) (cost + getRegionLoadCost(regionLoadList));
            }
        }

        // Add the total cost to the stats.
        // 计算每个RS的cost
        stats[i] = cost;
    }

    // Now return the scaled cost from data held in the stats object.
    // 计算整个集群的cost，和上面RegionCount算法一致。
    return costFromArray(stats);
}

protected double getRegionLoadCost(Collection<BalancerRegionLoad> regionLoadList) {
    double cost = 0;
    double previous = 0;
    boolean isFirst = true;
    for (BalancerRegionLoad rl : regionLoadList) {
        double current = getCostFromRl(rl);
        if (isFirst) {
            isFirst = false;
        } else {
            cost += current - previous;
        }
        previous = current;
    }
    return Math.max(0, cost / (regionLoadList.size() - 1));
}

// 关注点：Write、Read等负载信息是如何统计出来的
private synchronized void updateRegionLoad() {
    // We create a new hashmap so that regions that are no longer there are removed.
    // However we temporarily need the old loads so we can use them to keep the rolling average.
    Map<String, Deque<BalancerRegionLoad>> oldLoads = loads;
    loads = new HashMap<>();

    clusterStatus.getLiveServerMetrics().forEach((ServerName sn, ServerMetrics sm) -> {
        sm.getRegionMetrics().forEach((byte[] regionName, RegionMetrics rm) -> {
            String regionNameAsString = RegionInfo.getRegionNameAsString(regionName);
            Deque<BalancerRegionLoad> rLoads = oldLoads.get(regionNameAsString);
            if (rLoads == null) {
                rLoads = new ArrayDeque<>(numRegionLoadsToRemember + 1);
            } else if (rLoads.size() >= numRegionLoadsToRemember) {
                rLoads.remove();
            }
            // 更新这一次的RegionLoad
            rLoads.add(new BalancerRegionLoad(rm));
            loads.put(regionNameAsString, rLoads);
        });
    });

    for(CostFromRegionLoadFunction cost : regionLoadFunctions) {
        cost.setLoads(loads);
    }
}

// 疑问：RegionLoad多久上报一次
public class ClusterStatusChore extends ScheduledChore {
    private static final Logger LOG = LoggerFactory.getLogger(ClusterStatusChore.class);
    private final HMaster master;
    private final LoadBalancer balancer;

    public ClusterStatusChore(HMaster master, LoadBalancer balancer) {
        // 疑问解答：每1分钟上报一次metric指标
        super(master.getServerName() + "-ClusterStatusChore", master, master.getConfiguration().getInt(
            "hbase.balancer.statusPeriod", 60000));
        this.master = master;
        this.balancer = balancer;
    }

    @Override
    protected void chore() {
        try {
            balancer.setClusterMetrics(master.getClusterMetricsWithoutCoprocessor());
        } catch (InterruptedIOException e) {
            LOG.warn("Ignoring interruption", e);
        }
    }
}

// 通过代码分析，balance时的负载是取的最近15分钟的数据做的平均值。
```



| CostFunction                | 默认权重 | cost计算值 |
| --------------------------- | -------- | ---------- |
| RegionCountSkewCostFunction | 500      | 0.065      |
| MoveCostFunction            | 7        | x/600      |
| ServerLocalityCostFunction  | 25       |            |
| RackLocalityCostFunction    | 15       |            |
| TableSkewCostFunction       | 35       |            |
| ReadRequestCostFunction     | 5        |            |
| WriteRequestCostFunction    | 5        |            |
| MemStoreSizeCostFunction    | 5        |            |
| StoreFileCostFunction       | 5        |            |



#### 生成执行计划

![](../images/hbase/a30b0ece6a41efbfece74466cc2e48cefdfc9bf3.png)

所谓的执行计划，就是一些action集合。action的类型有：AssignRegionAction、MoveRegionAction和SwapRegionsAction三种。

而action由CandidateGenerator产生，核心代码如下：

```java
Cluster.Action nextAction(Cluster cluster) {
    return candidateGenerators.get(RANDOM.nextInt(candidateGenerators.size()))
        .generate(cluster);
}
```

其中，CandidateGenerator目前主要是以下几种：

![](../images/hbase/CandidateGenerator.png)

而在StochasticLoadBalancer中，使用了以下四种：

```java
if (this.candidateGenerators == null) {
    candidateGenerators = Lists.newArrayList();
    candidateGenerators.add(new RandomCandidateGenerator());
    candidateGenerators.add(new LoadCandidateGenerator());
    candidateGenerators.add(localityCandidateGenerator);
    candidateGenerators.add(new RegionReplicaRackCandidateGenerator());
}
```

**RandomCandidateGenerator**

1. 随机选择2个server，1个称为thisServer，另1个称为otherServer；
2. 对于region数量更多的那个server，随机选出1个region；
3. 对于region数量更少的那个server，有50%概率选出0个region，有50%概率选出1个region；
4. 如果第3步选出了0个region，则返回MoveRegionAction，否则返回SwapRegionsAction；

**LoadCandidateGenerator**

1. 将所有server按regionCount排序；
2. 找出region最多和最少的2个server；
3. 后续步骤与RandomCandidateGenerator一致；

**LocalityBasedCandidateGenerator**

1. 将所有region打乱顺序；
2. 遍历这些region；
3. 获取当前这个region(称为fromRegion)的mostLocalServer(称为toServer)，如果与当前所在server(称为fromServer)一致，则回到第2步，否则到下一步；
4. 如果toServer的region数量小于平均值，则直接返回一个MoveRegionAction，否则到下一步；
5. 遍历toServer上的region(称为toRegion)，如果能找到一个region，与fromRegion交换可使得总体locality增加，则返回一个SwapRegionsAction，否则回到第2步；



## 业务分析

在HBase的CostFunction中可以看出，RegionCount权重很高。它的目的主要是为了让每个RS中管理的Region数尽可能均匀。其余的Cost用来辅助计算。

Balance计算公式：cost = sum[f(cost) * multiplier] / sum[multiplier] < 0.05

cost = (f(RegionCountSkewCostFunction) * 500 + f(MoveCostFunction) * 7 + f(ServerLocalityCostFunction) * 25 + f(RackLocalityCostFunction) * 15 + f(TableSkewCostFunction) * 35 + f(ReadRequestCostFunction) * 5  + f(WriteRequestCostFunction) * 5 + f(MemStoreSizeCostFunction) * 5 + f(StoreFileCostFunction) * 5) / ( 500 + 7 + 25 + 15 + 35 +20)

### 场景分析

**1. 单个RS宕机重启后无法分配Region，是什么原因**

在上面分析的CostFunction中，RegionCountCostFunction权重最高。

默认容忍的不均衡比率为0.05，当小于0.05时，不触发balance。0.05 * 602 / 500 = 0.0602

即：如果RegionCountCostFunction的cost值小于0.0602时，很有可能不进行balance。

和平均值差异个数 / max > 0.0602

以线上2000个Region为例。max = 3750。即：与平均值差异达到226以上，才可能进行balance。

2000个Region，分到16个RS上，平均值为125。分到15个RS上，平均值为133。

因此，启动一个RS后，差异值为：125 + 8 * 15 = 245





**2. 假设其它都完全平衡，Write差异多少会触发Balance**

设：集群中WriteRequestCost值为 x ；

x * 5 / 602 > 0.05

x > 6.02

则：集群中，WriteRequestCost 大于 6.02 时才会进行balance。

因为每个cost的函数取值范围为[0, 1]，即：假使每个RS的Region都分布很均匀，但其中一个RS负责所有的写。那么，也不再进行平衡。

也就是说，RegionCountSkewCostFunction做决策，WriteRequestCostFunction做协助。是否进行balance，由RegionCountSkewCostFunction决定，在进行balance时，可以通过WriteRequestCostFunction让它尽可能最佳。



**3. 如果把其它CostFunction都禁用，只保留RegionCount和WriteRequest会咋样**

如果把其它的都禁用，重启整个HBase，理论上会出现，Region在RS上分布均匀，且每个RS的写基本均衡。

带来的影响是：Table分布可能不均、Region文件本地化可能不够平衡、HFile可能会不均衡（在我们业务场景下不会出现）。

Table分布不均带来的影响是：如果某个table分布在其中一个RS上，这个RS出现繁忙时，整个table的操作性能也会跟着下降。（以我们当前的业务场景，核心表就只有一个，故，它的影响基本可以忽略）。

Region文件本地化不均衡带来的影响：查询变慢。



**4. 增大WriteRequest权重会怎样**

如果以写为主的集群，增加WriteRequest的权重，会优先根据写流量进行平衡。是否该进行balance，决策权由写流量控制。每个RS上分布的Region数量可能会存在一点差异，不完全平均。



**5. 我们的业务环境下该如何配置**

我们是写多读少型，且大部分Region处于不活跃状态。因此，我们业务下的权重分配应该是：WriteRequest高权重，RegionCount次之，其它保持不变。

WriteRequest和RegionCount分别配置多少合理？

RegionCount的默认权重是500，因此，WriteRequest的权重也保持为500。

高WriteRequest权重下带来的影响：

1. 补数据或跑rollup时，会有写波动。它们是按Region跑的。会带来频繁的balance。

为解决这个问题，可以让balance时间由5分钟触发一次变为1小时触发一次。

RegionCount权重给多少合理呢？

而在我们的业务场景下，有大部分Region是闲置的。即使限制，也要尽可能均匀的分配在每个RS上，因为一旦出现大范围历史查询时，会对某个RS造成压力。

假设WriteRequest均衡时，RegionCount差异存在10%以上时，权重达到多少才能进行balance？

x * 0.1 / x + 597 > 0.05   ==>  x > 597  已超过WriteRequest权重。

假设RegionCount存在20%以上差异时，权重需要高于199。当RegionCount存在30%以上差异时，权重需要高于119.4。

可以将RegionCount权重设置为120。此时，WriteRequest能容忍的差异为：y * 500 / 717 > 0.05  y>0.0717

假设总的写入量为70K。max = 70/15 * 13 + 70 = 130.67，总流量差异为：9.36K。即，平均每个RS与平均的写入流量差异为：0.6K。



## 业务验证

### 场景构建

线下环境只有3个RS。

第一步：构建一个共90个Region的表，将WriteRequest权重设置为500，RegionCount权重设置为120。

第二步：初次启动HBase的表象应该为：每个RS负责30个Region。因为没有写请求，WriteRequestCost为0。

第三步：找出某个RS上的所有Region，为这些Region上发数据，总请求量为15K。此时的现象为：单个RS负责15K写请求，其余两个RS为0。

第四步：5分钟后自动触发balance，此时的现象应该为：每个RS写请求近似均匀5K，总差异不超过：1.4K，RegionCount也近似均匀30。



### 实际验证

第一步：生成split file。

```sh
vim split
内容：
1
2
……
88
89
```

第二步：创建带预分区的测试表

```sh
hbase> create 'balance_test', 't', SPLITS_FILE=>'/home/lizy/split'
```

第三步：更改hbase配置后重启

![](../images/hbase/20220606145148.jpg)

![](../images/hbase/20220606145303.jpg)

第四步：找出某个RS下的所有Region进行写操作

以bigdata-master-99为例：

![](../images/hbase/20220606145715.jpg)

第五步：写一段时间后，查看Balance日志及balance结果

![](../images/hbase/20220606165150.jpg)

![](../images/hbase/1654505979702.jpg)

通过日志分析：执行了3次balance后，写平衡后，再没执行过balance。

第六步：模拟补数据场景

在步骤五的基础上，再增加一个写操作。只不过该写操作是按Region写的。每个Region写30s，换下一个Region。30个Region持续写15分钟，看整体效果。

![](../images/hbase/20220607105453.jpg) ![](../images/hbase/20220607105608.jpg)



在进行补数据的过程中，发生了多次balance，补完以后，其中一个RS无写入，再balance之后才平衡。

意味着，在进行补数据时，单纯的调整costFunction权重是不够的。

第七步：将balance时间间隔设置为1小时，balance时用到的numRegionLoadsToRemember由15调整到60。目的：补数据、rollup等操作在短时间内是不平衡的，但是在小时级别下，基本是均匀的。除非补历史大范围全量数据（一个任务跑几天的）。

![](../images/hbase/20220607110901.jpg)

第八步：重启HBase，重复以上操作

![](../images/hbase/20220607121036.jpg)

搜索Balance日志，并没看到执行了balance。

![](../images/hbase/20220607121226.jpg)





参考文献：

[HBase balancer原理解析和一些优化方案](https://blog.csdn.net/ihaxiaolin/article/details/122984712)

[HBase源码分析4—Region Balance](https://blog.yiz96.com/hbase-src-4-region-balance/)

[balancer-爱代码爱编程](https://icode.best/i/08442445750231)