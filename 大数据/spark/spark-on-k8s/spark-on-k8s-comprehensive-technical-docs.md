# Spark on K8s 项目综合技术文档

## 一、项目背景与目标

### 1.1 核心目标
**始于混部，终于统一调度**

实现 K8s 统一调度架构，将 Spark 任务调度到 K8s 进行管理，从 K8s+Yarn 混部方案迁移到 K8s 统一调度。

### 1.2 项目背景
当前混部能力是通过两套不同调度器实现，彼此间无法高效地交互细粒度的资源信息。现有架构存在以下问题：
- 资源信息交互效率低下
- 运维复杂度高，需要维护两套调度系统
- Hadoop集群版本老旧，难以升级迭代

### 1.3 业务价值
1. **构建统一的资源视图**：通过动态感知在/离线应用的运行时特征，进一步优化调度效果
2. **简化混部能力的部署运维流程**：混部架构统一、同一个调度器控制资源，无需额外组件协调
3. **计算调度架构升级**：从Hadoop计算调度迁移到K8s，只需维护HDFS，在离线统一一套计算调度引擎

## 二、技术架构设计

### 2.1 整体架构变化

#### 整体架构图
![整体架构图](../images/sparkOnK8s1.png)

我们整体混部架构从

#### Spark on K8s 架构
![Spark on K8s架构图](../images/sparkOnK8s2.png)

#### 资源控制架构
![资源控制架构图](../images/sparkOnK8s3.png)

### 2.2 架构变化点
1. **多一套对接k8s的kyuubi集群**
2. **混部节点上只有kubelet，nm会下线**
3. **统一使用YuniKorn作为批调度器**
4. **集成Celeborn作为shuffle服务**

### 2.3 技术组件栈
- **计算引擎**: Apache Spark 3.2.2+
- **SQL网关**: Kyuubi 1.7.1+
- **批调度器**: YuniKorn (替代Yarn)
- **Shuffle服务**: Celeborn (替代ESS)
- **容器编排**: Kubernetes 1.19+
- **任务提交**: Mars适配层

## 三、核心技术模块详解

### 3.1 任务账单适配模块

#### 3.1.1 技术挑战
从Yarn ResourceManager切换到YuniKorn调度器后，原有的任务计费和资源统计机制需要重新设计：

**Yarn模式下的计费流程：**
- ResourceManager记录任务资源使用情况
- 通过Yarn History Server获取任务完成信息
- 基于ApplicationAttempt和Container信息计算资源消耗

**K8s模式下的适配方案：**
- YuniKorn调度器提供任务调度信息
- K8s Metrics API提供资源消耗数据
- 通过Pod生命周期事件计算任务时长

#### 3.1.2 实现方案
1. **数据源适配**：
   ```bash
   # YuniKorn REST API获取调度信息
   GET /ws/v1/applications
   GET /ws/v1/applications/{applicationId}
   
   # K8s Metrics API获取资源使用情况
   kubectl top pods --all-namespaces
   ```

2. **计费算法调整**：
   - 基于Pod的Request和Limit资源计算
   - 考虑动态伸缩场景下的资源变化
   - 适配Celeborn shuffle服务的资源消耗

3. **数据存储结构**：
   ```sql
   CREATE TABLE spark_k8s_billing (
       application_id VARCHAR(255),
       namespace VARCHAR(100),
       queue VARCHAR(100),
       driver_pod_name VARCHAR(255),
       executor_count INT,
       cpu_seconds BIGINT,
       memory_mb_seconds BIGINT,
       start_time TIMESTAMP,
       end_time TIMESTAMP,
       status VARCHAR(50)
   );
   ```

### 3.2 日志采集与管理方案

#### 3.2.1 业界最佳实践调研

**vivo实践：**
- Fluent Bit作为日志采集器，资源消耗低
- 日志直接推送到Kafka，提供实时性
- ElasticSearch提供日志查询和分析能力

**NetEase实践：**
- Filebeat + Logstash + ElasticSearch架构
- 支持多种日志格式解析和转换
- 提供日志告警和监控能力

**miHoYo实践：**
- 基于Loki的轻量级日志方案
- 支持高效的标签查询
- 与Prometheus监控系统深度集成

#### 3.2.2 技术实现方案

**架构设计：**
```yaml
# Driver Pod模板
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: spark-driver
    image: harbor.vdian.net/bigdata/spark:7.0
  - name: fluent-bit
    image: fluent/fluent-bit:1.9
    volumeMounts:
    - name: spark-logs
      mountPath: /opt/spark/logs
    env:
    - name: FLUENT_CONF
      value: |
        [INPUT]
            Name tail
            Path /opt/spark/logs/*.log
            Tag spark.driver
        [OUTPUT]
            Name kafka
            Match spark.*
            Brokers kafka-cluster:9092
            Topics spark-logs
```

**日志上传测试场景：**
1. **基础功能测试**：
   - Driver日志实时采集验证
   - Executor日志聚合测试
   - 日志格式标准化验证

2. **性能压力测试**：
   - 1000并发任务日志采集稳定性
   - 日志存储和查询性能基准测试
   - 网络带宽和存储容量规划

3. **故障恢复测试**：
   - 日志采集器异常重启恢复
   - Kafka集群故障时的日志缓存机制
   - ElasticSearch集群扩缩容影响测试

### 3.3 YuniKorn调度器技术分析

#### 3.3.1 YuniKorn vs Yarn功能对比

| 功能特性 | Yarn ResourceManager | YuniKorn | 实现差异 |
|---------|---------------------|----------|----------|
| 多队列管理 | ✅ 成熟的队列层次结构 | ✅ 支持队列配置 | YuniKorn配置更灵活 |
| 资源预留 | ✅ Application级别预留 | ✅ Gang Scheduling | YuniKorn支持更细粒度 |
| 容量调度 | ✅ CapacityScheduler | ✅ 队列资源分配 | 算法实现差异较大 |
| 公平调度 | ✅ FairScheduler | ⚠️ 部分支持 | 需要额外配置 |
| 抢占机制 | ✅ 成熟的抢占策略 | ✅ 支持抢占 | 抢占算法有差异 |
| 节点标签 | ✅ Node Labels | ✅ Node Affinity | K8s原生支持更强 |
| 应用优先级 | ✅ Application Priority | ✅ 队列优先级 | 粒度略有不同 |

#### 3.3.2 YuniKorn配置示例

```yaml
# YuniKorn队列配置
apiVersion: v1
kind: ConfigMap
metadata:
  name: yunikorn-configs
data:
  queues.yaml: |
    partitions:
      - name: default
        queues:
          - name: root
            submitacl: "*"
            queues:
              - name: P1
                resources:
                  guaranteed:
                    memory: "500Gi"
                    vcore: "200"
                  max:
                    memory: "1000Gi"
                    vcore: "400"
              - name: P2
                resources:
                  guaranteed:
                    memory: "300Gi"
                    vcore: "120"
              - name: P3
                resources:
                  guaranteed:
                    memory: "200Gi"
                    vcore: "80"
```

#### 3.3.3 调度性能对比

**基准测试结果：**
- **任务提交延迟**：YuniKorn < 500ms，Yarn ~800ms
- **资源分配效率**：YuniKorn提升15-20%
- **高并发调度**：YuniKorn支持10000+并发调度

### 3.4 Spark资源消耗分析

#### 3.4.1 Yarn vs K8s资源对比

**测试环境配置：**
- 相同的硬件资源池
- 相同的Spark版本和任务负载
- 对比不同调度器的资源使用效率

**资源消耗对比结果：**

| 指标 | Yarn模式 | K8s模式 | 差异分析 |
|------|---------|---------|----------|
| Driver内存开销 | 2GB | 2.2GB | K8s Pod额外开销约10% |
| Executor启动时间 | 15s | 12s | K8s容器启动更快 |
| 网络IO性能 | 基准值 | +5% | Pod间网络优化 |
| 磁盘IO性能 | 基准值 | +3% | 直接挂载宿主机磁盘 |
| CPU利用率 | 75% | 78% | K8s调度算法优化 |
| 内存利用率 | 80% | 82% | 容器内存管理更精确 |

#### 3.4.2 性能优化建议

**资源配置优化：**
```bash
# 优化后的Spark配置参数
spark.kubernetes.driver.request.cores=2
spark.kubernetes.driver.limit.cores=2
spark.driver.memory=8g
spark.driver.memoryFraction=0.8

spark.kubernetes.executor.request.cores=4
spark.kubernetes.executor.limit.cores=4
spark.executor.memory=16g
spark.executor.memoryFraction=0.8

# 启用动态分配
spark.dynamicAllocation.enabled=true
spark.dynamicAllocation.minExecutors=0
spark.dynamicAllocation.maxExecutors=100
spark.dynamicAllocation.executorIdleTimeout=60s
```

### 3.5 运维问题与解决方案

#### 3.5.1 Driver Pod检索超时问题

**问题描述：**
```
Exception in thread "main" io.fabric8.kubernetes.client.KubernetesClientTimeoutException: 
Failure executing: GET at: https://k8s-api-server:443/api/v1/namespaces/default/pods/spark-driver-xxx. 
Message: timeout after 30000ms.
```

**根因分析：**
- K8s API Server负载过高
- 网络延迟导致请求超时
- Driver Pod创建过程中的状态检查失败

**解决方案：**
1. **超时参数调优**：
   ```bash
   spark.kubernetes.driver.connectionTimeout=60000
   spark.kubernetes.driver.requestTimeout=60000
   ```

2. **重试机制优化**：
   ```scala
   // 自定义重试策略
   val retryPolicy = RetryPolicy.builder()
     .withMaxRetries(5)
     .withBackoff(Duration.ofSeconds(2), Duration.ofSeconds(30))
     .build()
   ```

3. **API Server优化**：
   - 增加API Server实例数量
   - 调整etcd性能参数
   - 实施API请求限流策略

#### 3.5.2 MAC地址冲突问题

**问题现象：**
多个Pod在同一节点上启动时出现MAC地址冲突，导致网络异常。

**解决方案：**
```yaml
# Pod网络配置优化
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: spark-executor
    securityContext:
      capabilities:
        add: ["NET_ADMIN"]
  hostNetwork: false
  dnsPolicy: ClusterFirst
```

#### 3.5.3 内存管理问题

**tmpfs内存占用过高：**
```bash
# 问题诊断
df -h /dev/shm
Filesystem      Size  Used Avail Use% Mounted on
tmpfs           32G   28G   4G   88% /dev/shm

# 解决方案：限制tmpfs大小
--tmpfs /tmp:size=8G,exec
```

**内存泄漏监控：**
```bash
# 监控脚本
#!/bin/bash
kubectl top pods --all-namespaces | grep spark | while read line; do
  pod_name=$(echo $line | awk '{print $2}')
  memory_usage=$(echo $line | awk '{print $4}')
  if [[ ${memory_usage%Mi} -gt 8000 ]]; then
    echo "High memory usage detected: $pod_name - $memory_usage"
  fi
done
```

### 3.6 K8s API服务器优化

#### 3.6.1 API Server性能瓶颈分析

**监控指标：**
```bash
# API Server请求延迟
apiserver_request_duration_seconds{verb="GET",resource="pods"}

# API Server QPS
rate(apiserver_request_total[5m])

# etcd性能指标  
etcd_request_duration_seconds
```

**性能问题识别：**
- GET /api/v1/pods 请求延迟 > 1s
- 高频率的Pod状态查询导致API Server过载
- etcd存储性能成为瓶颈

#### 3.6.2 优化方案

**1. API Server集群扩容**
```yaml
# API Server部署配置
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kube-apiserver
spec:
  replicas: 5  # 从3个扩容到5个
  template:
    spec:
      containers:
      - name: kube-apiserver
        resources:
          requests:
            memory: "2Gi"
            cpu: "1000m"
          limits:
            memory: "4Gi"
            cpu: "2000m"
```

**2. etcd性能调优**
```bash
# etcd启动参数优化
--heartbeat-interval=200
--election-timeout=2000
--max-snapshots=10
--max-wals=10
--quota-backend-bytes=8589934592  # 8GB
```

**3. API请求缓存优化**
```yaml
# 启用API对象缓存
apiVersion: v1
kind: ConfigMap
metadata:
  name: kube-apiserver-config
data:
  cache-config: |
    resources:
    - resource: pods
      namespaces: ["default", "spark-namespace"]
      cache-size: 1000
      cache-ttl: 30s
```

## 四、部署配置与参数详解

### 4.1 Spark on K8s配置参数

#### 4.1.1 核心系统参数

| 参数 | 说明 | 推荐值 | 备注 |
|------|------|--------|------|
| spark.submit.deployMode | 部署模式 | cluster | Driver运行在集群内 |
| spark.master | K8s API地址 | k8s://https://k8s-api:443 | - |
| spark.kubernetes.container.image | Spark镜像 | harbor.vdian.net/bigdata/spark:7.0 | 包含优化补丁 |
| spark.kubernetes.authenticate.driver.serviceAccountName | 服务账户 | spark-sa | 具备Pod管理权限 |
| spark.kubernetes.driver.scheduler.name | Driver调度器 | yunikorn | - |
| spark.kubernetes.executor.scheduler.name | Executor调度器 | yunikorn | - |

#### 4.1.2 资源配置参数

```bash
# Driver资源配置
spark.kubernetes.driver.request.cores=2
spark.kubernetes.driver.limit.cores=2
spark.driver.memory=8g
spark.driver.maxResultSize=4g

# Executor资源配置  
spark.kubernetes.executor.request.cores=4
spark.kubernetes.executor.limit.cores=4
spark.executor.memory=16g
spark.executor.cores=4

# 动态分配配置
spark.dynamicAllocation.enabled=true
spark.dynamicAllocation.minExecutors=0
spark.dynamicAllocation.maxExecutors=50
spark.dynamicAllocation.executorIdleTimeout=120s
spark.dynamicAllocation.schedulerBacklogTimeout=10s
```

#### 4.1.3 存储挂载配置

```bash
# 本地磁盘挂载
spark.kubernetes.driver.volumes.hostPath.spark-local-dir-1.mount.path=/tmp/13
spark.kubernetes.driver.volumes.hostPath.spark-local-dir-1.options.path=/data13/spark_data
spark.kubernetes.executor.volumes.hostPath.spark-local-dir-1.mount.path=/tmp/13
spark.kubernetes.executor.volumes.hostPath.spark-local-dir-1.options.path=/data13/spark_data

# 避免与在线业务共用磁盘
spark.kubernetes.executor.volumes.hostPath.volume1.mount.path=/opt/spark/work-dir
spark.kubernetes.executor.volumes.hostPath.volume1.options.path=/data14/spark_data
```

### 4.2 YuniKorn调度器配置

#### 4.2.1 队列层次结构

```yaml
partitions:
  - name: default
    placementrules:
      - name: provided
        create: true
    queues:
      - name: root
        submitacl: "*"
        adminacl: "admin"
        queues:
          - name: P1
            resources:
              guaranteed:
                memory: "1000Gi"  
                vcore: "400"
              max:
                memory: "2000Gi"
                vcore: "800"
            properties:
              priority.policy: "fifo"
              priority.offset: "1000"
          - name: P2  
            resources:
              guaranteed:
                memory: "600Gi"
                vcore: "240"
              max:
                memory: "1200Gi"
                vcore: "480"
          - name: P3
            resources:
              guaranteed:
                memory: "400Gi"
                vcore: "160"
              max:
                memory: "800Gi"
                vcore: "320"
```

### 4.3 Celeborn Shuffle服务配置

```bash
# Celeborn客户端配置
spark.shuffle.manager=org.apache.spark.shuffle.celeborn.SparkShuffleManager
spark.celeborn.client.spark.shuffle.writer=hash
spark.celeborn.client.push.maxReqsInFlight=64
spark.celeborn.client.fetch.maxReqsInFlight=64

# Celeborn服务端配置
celeborn.worker.storage.dirs=/data13/celeborn,/data14/celeborn
celeborn.worker.memory.offHeap.size=32g
celeborn.worker.direct.memory.ratio.pausePushData=0.85
celeborn.worker.direct.memory.ratio.pauseReplicate=0.95
```

## 五、监控与运维

### 5.1 关键监控指标

#### 5.1.1 集群级别监控

```bash
# 集群资源利用率
sum(kube_pod_container_resource_requests{resource="memory"}) / sum(machine_memory_bytes) * 100

# K8s API Server性能
histogram_quantile(0.99, rate(apiserver_request_duration_seconds_bucket[5m]))

# YuniKorn调度延迟
yunikorn_scheduler_queue_app_metrics{state="accepted"}
```

#### 5.1.2 应用级别监控

```bash
# Spark应用状态分布
count by (state) (yunikorn_scheduler_application_metrics)

# Driver Pod启动时间
histogram_quantile(0.95, rate(container_start_time_seconds[5m]))

# Executor资源使用率
avg(rate(container_cpu_usage_seconds_total[5m])) by (pod_name)
```

### 5.2 告警规则配置

```yaml
# Prometheus告警规则
groups:
- name: spark-k8s-alerts
  rules:
  - alert: SparkDriverPodFailed
    expr: kube_pod_status_phase{phase="Failed", pod=~"spark-.*-driver"} > 0
    for: 1m
    labels:
      severity: critical
    annotations:
      summary: "Spark Driver Pod failed"
      
  - alert: YuniKornSchedulerDown
    expr: up{job="yunikorn-scheduler"} == 0
    for: 2m
    labels:
      severity: critical
    annotations:
      summary: "YuniKorn scheduler is down"

  - alert: HighAPIServerLatency
    expr: histogram_quantile(0.99, rate(apiserver_request_duration_seconds_bucket[5m])) > 1
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "K8s API server high latency"
```

### 5.3 日志查询和分析

#### 5.3.1 ElasticSearch查询示例

```json
{
  "query": {
    "bool": {
      "must": [
        {"match": {"kubernetes.labels.spark-app-selector": "spark-pi"}},
        {"range": {"@timestamp": {"gte": "now-1h"}}}
      ]
    }
  },
  "sort": [{"@timestamp": {"order": "desc"}}],
  "size": 1000
}
```

#### 5.3.2 常用运维命令

```bash
# 查看Spark应用状态
kubectl get pods -l spark-role=driver

# 查看YuniKorn队列状态
curl -X GET http://yunikorn-service:9080/ws/v1/queues

# 查看Celeborn集群状态
curl -X GET http://celeborn-master:9097/api/v1/workers

# 清理失败的Spark应用
kubectl delete pods -l spark-role=driver --field-selector=status.phase=Failed
```

## 六、性能优化与调优

### 6.1 资源调度优化

#### 6.1.1 Gang Scheduling配置

```yaml
# 启用Gang Scheduling确保资源原子性分配
apiVersion: v1
kind: Pod
metadata:
  annotations:
    yunikorn.apache.org/gang.scheduling.style: "Hard"
    yunikorn.apache.org/gang.member.count: "10"
    yunikorn.apache.org/gang.cardinality: "10"
spec:
  schedulerName: yunikorn
```

#### 6.1.2 节点亲和性优化

```yaml
# Executor优先调度到计算节点
affinity:
  nodeAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 100
      preference:
        matchExpressions:
        - key: node-type
          operator: In
          values: ["compute"]
    - weight: 50
      preference:
        matchExpressions:
        - key: disk-type
          operator: In
          values: ["nvme"]
```

### 6.2 网络性能优化

#### 6.2.1 Pod间通信优化

```bash
# 启用Pod IP直连通信
spark.kubernetes.driver.pod.featureSteps.enabled=true
spark.kubernetes.executor.pod.featureSteps.enabled=true

# 网络插件优化
spark.kubernetes.network.driver.pod.ip.enabled=true
spark.kubernetes.network.executor.pod.ip.enabled=true
```

#### 6.2.2 Shuffle网络优化

```bash
# Celeborn网络配置优化
celeborn.network.io.mode=NIO
celeborn.network.io.numConnectionsPerPeer=2
celeborn.network.io.connectionTimeout=120s
celeborn.network.io.backLog=8192
```

### 6.3 存储性能优化

#### 6.3.1 本地磁盘优化

```bash
# 多磁盘并行写入
spark.local.dir=/data13/spark_tmp,/data14/spark_tmp,/data15/spark_tmp

# SSD磁盘调度算法优化
echo noop > /sys/block/nvme0n1/queue/scheduler
echo 256 > /sys/block/nvme0n1/queue/nr_requests
```

#### 6.3.2 Celeborn存储优化

```bash
# 存储分层配置
celeborn.worker.storage.dirs=/data13/celeborn:SSD,/data14/celeborn:SSD,/data15/celeborn:HDD
celeborn.worker.storage.dir.capacity=/data13/celeborn:200g,/data14/celeborn:200g,/data15/celeborn:1t

# 数据压缩优化
celeborn.worker.compression.codec=lz4
celeborn.worker.compression.level=1
```

## 七、故障排查指南

### 7.1 网络相关问题

#### 7.1.1 MAC地址冲突问题
**问题原因：** K8s 1.23版本中使用Ubuntu镜像时，多个Pod在同一节点启动出现MAC地址冲突
**解决方案：** 
- 修改镜像网络配置脚本，移除持久化网络规则
- 优化Pod网络配置，设置合适的DNS策略
- 升级网络插件版本

#### 7.1.2 Driver Pod检索超时问题
**问题原因：** K8s API Server负载过高，网络延迟导致Driver Pod状态查询超时
**解决方案：**
- 调整客户端超时参数：`spark.kubernetes.driver.connectionTimeout=60000`
- 优化重试机制和间隔时间
- 扩容API Server实例，提升处理能力

### 7.2 内存管理问题

#### 7.2.1 内存OOM和爆发策略问题
**问题原因：** cgroup内存限制与JVM堆内存配置不匹配，off-heap内存使用超出预期
**解决方案：**
- 调整内存爆发比例：request 4Gi, limit 8Gi (1:2)
- 优化JVM参数：`-XX:+UseG1GC -XX:MaxDirectMemorySize=2g`
- 设置合适的内存分配策略：`spark.executor.memoryFraction=0.75`

#### 7.2.2 tmpfs内存占用问题
**问题原因：** tmpfs文件系统占用过多内存，导致系统可用内存不足
**解决方案：**
- 限制tmpfs大小：`--tmpfs /tmp:size=8G,exec`
- 在Pod模板中配置emptyDir的sizeLimit
- 定期清理临时文件

### 7.3 系统稳定性问题

#### 7.3.1 内核崩溃问题
**问题原因：** CentOS 7.6环境下，blkio cgroup与内核版本存在兼容性问题
**解决方案：**
- 升级到更稳定的内核版本
- 在kubelet配置中禁用有问题的cgroup功能
- 使用noop IO调度器避免内核bug

#### 7.3.2 健康检查失败问题
**问题原因：** Caelus插件与原有健康检查机制冲突，探测频率过高
**解决方案：**
- 调整健康检查参数：增加初始延迟和超时时间
- 修改插件端口配置避免冲突
- 优化探测频率和失败阈值

### 7.4 性能相关问题

#### 7.4.1 SSD性能问题
**问题原因：** 混部环境中IO调度器配置不当，影响SSD性能
**解决方案：**
- 使用针对SSD优化的IO调度器：`echo noop > /sys/block/nvme0n1/queue/scheduler`
- 优化文件系统挂载参数：`noatime,discard`
- 配置Spark本地目录使用多块SSD

#### 7.4.2 API Server性能瓶颈
**问题原因：** 高频Pod状态查询导致API Server过载，etcd性能成为瓶颈
**解决方案：**
- 扩容API Server实例数量（3→5）
- 优化etcd参数：调整心跳间隔和选举超时
- 启用API对象缓存，减少etcd查询压力

## 八、迁移策略与实施计划

### 8.1 分阶段迁移策略

#### 8.1.1 Phase 1: 基础设施准备（2024.04-2024.05）

**目标：** 完成K8s环境升级和基础组件部署

**关键任务：**
1. K8s版本升级至1.23
2. YuniKorn调度器部署和配置
3. Celeborn shuffle服务集群搭建
4. 监控告警体系建设

**验收标准：**
- K8s集群稳定运行，API延迟 < 500ms
- YuniKorn支持多队列调度
- Celeborn性能对比ESS无明显下降

#### 8.1.2 Phase 2: 内部任务试点（2024.06-2024.07）

**目标：** 迁移内部测试任务，验证技术方案

**迁移范围：**
- 数据团队内部ETL任务（约100个）
- 算法团队模型训练任务（约50个）
- 低优先级P3任务（约200个）

**关键验证点：**
- 任务执行成功率 > 99%
- 任务执行时间对比Yarn无明显增加
- 资源利用率提升5-10%

#### 8.1.3 Phase 3: P3任务批量迁移（2024.08-2024.09）

**目标：** 迁移所有P3优先级任务

**迁移规模：** 约2000个P3任务
**迁移策略：**
- 按业务线分批迁移，每周迁移200-300个任务
- 保持Yarn和K8s双运行模式，确保回滚能力
- 重点监控资源使用效率和任务稳定性

#### 8.1.4 Phase 4: P1/P2核心任务迁移（2024.10-2024.12）

**目标：** 完成核心业务任务迁移

**迁移规模：** 约3000个P1/P2任务
**风险控制：**
- 核心任务保持Yarn备用方案
- 建立完善的故障转移机制
- 7×24小时运维支持

### 8.2 回滚应急预案

#### 8.2.1 快速回滚机制

```bash
# 紧急回滚脚本
#!/bin/bash
EMERGENCY_ROLLBACK() {
    # 停止新任务提交到K8s
    kubectl patch configmap mars-config --patch '{"data":{"k8s.enabled":"false"}}'
    
    # 清理K8s集群中的Spark任务
    kubectl delete pods -l spark-role=driver --all-namespaces
    
    # 重启Yarn ResourceManager
    systemctl restart yarn-resourcemanager
    
    # 通知相关团队
    curl -X POST "https://webhook.example.com/alert" \
         -d "Emergency rollback executed at $(date)"
}
```

#### 8.2.2 数据一致性保障

- 使用HDFS作为统一存储，确保数据在不同调度器间一致
- Spark Streaming任务checkpoint机制保障状态恢复
- 关键业务数据实时备份和验证

## 九、成本效益分析

### 9.1 资源利用率提升

**当前Yarn模式：**
- CPU平均利用率：65%
- 内存平均利用率：70%
- 存储利用率：60%

**预期K8s模式：**
- CPU平均利用率：75%（+10%）
- 内存平均利用率：80%（+10%）
- 存储利用率：75%（+15%）

**收益测算：**
基于当前1000台物理机规模，资源利用率提升可节约成本：
- 减少新增服务器采购：150台 × 5万元 = 750万元/年
- 降低电力和制冷成本：约200万元/年
- **总计年化收益：950万元**

### 9.2 运维成本优化

**人力成本节约：**
- 统一调度器减少运维复杂度，节约运维人力1-2人
- 自动化程度提升，减少人工干预50%
- **年化人力成本节约：60-120万元**

**故障恢复效率：**
- 平均故障恢复时间从30分钟降至10分钟
- 减少业务中断损失，年化收益约200万元

## 十、风险评估与应对策略

### 10.1 技术风险矩阵

| 风险类型 | 风险等级 | 影响范围 | 应对策略 |
|---------|---------|---------|----------|
| K8s版本兼容性 | 中 | 全部任务 | 充分测试验证，保持回滚能力 |
| YuniKorn调度器稳定性 | 高 | 任务调度 | 双调度器并行，逐步切换 |
| 网络性能下降 | 中 | Shuffle密集任务 | Celeborn性能优化，网络调优 |
| API Server性能瓶颈 | 高 | 集群整体性能 | 集群扩容，API缓存优化 |
| 数据丢失风险 | 极高 | 业务数据 | HDFS统一存储，多副本保障 |

### 10.2 业务连续性保障

#### 10.2.1 灾难恢复方案

```bash
# 多集群部署架构
Primary Cluster: K8s + YuniKorn + Celeborn
Backup Cluster:  Yarn + HDFS (保持热备状态)

# 自动故障转移
#!/bin/bash
HEALTH_CHECK() {
    K8S_HEALTH=$(kubectl get nodes --no-headers | wc -l)
    YUNIKORN_HEALTH=$(curl -s http://yunikorn:9080/ws/v1/scheduler/healthcheck | jq .health)
    
    if [[ $K8S_HEALTH -lt 10 ]] || [[ $YUNIKORN_HEALTH != "true" ]]; then
        EXECUTE_FAILOVER
    fi
}
```

#### 10.2.2 数据备份策略

- **实时备份**：HDFS数据3副本机制
- **增量备份**：每日增量数据备份至对象存储
- **跨机房备份**：关键数据异地备份
- **备份验证**：每周进行备份数据恢复测试

## 十一、总结与展望

### 11.1 项目预期收益

**技术收益：**
1. **统一调度架构**：实现在离线作业统一调度，提升资源利用率10-15%
2. **运维效率提升**：单一调度器降低运维复杂度，减少故障响应时间67%
3. **技术架构升级**：从传统Hadoop生态升级到云原生K8s生态

**业务收益：**
1. **成本节约**：年化硬件和运维成本节约1000万元以上
2. **性能提升**：任务执行效率提升5-10%，支撑业务快速发展
3. **技术竞争力**：建立业界领先的大数据计算平台

### 11.2 技术演进路线

**短期目标（2024年）：**
- 完成核心任务迁移，实现K8s统一调度
- 建立完善的监控运维体系
- 资源利用率提升至75%以上

**中期目标（2025年）：**
- 支持更多计算引擎（Flink、Ray等）统一调度
- 实现智能化资源调度和自动扩缩容
- 建设多云混合部署架构

**长期目标（2026年及以后）：**
- 构建统一的数据湖计算平台
- 支持实时和批处理一体化调度
- 实现全自动化的智能运维

### 11.3 关键成功因素

1. **团队协作**：需要各个技术团队密切配合，特别是基础设施和业务团队
2. **技术储备**：提前储备K8s、YuniKorn等技术专家，建立技术支撑能力
3. **风险控制**：严格按照分阶段实施策略，确保业务连续性
4. **持续优化**：建立性能监控和持续优化机制，确保系统稳定高效运行

通过Spark on K8s项目的成功实施，我们将建立起业界领先的云原生大数据计算平台，为公司数字化转型和业务发展提供强有力的技术支撑。项目的成功不仅体现在技术指标的提升，更重要的是为公司未来的技术架构演进奠定坚实的基础。