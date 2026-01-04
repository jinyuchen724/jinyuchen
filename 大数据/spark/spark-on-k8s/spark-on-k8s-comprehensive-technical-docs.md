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

我们整体混部架构从yarn(离线任务)+k8s(在线业务)--->yarn(离线任务)+k8s(在线业务+离线业务)
最终态:k8s(在线业务+离线业务)

#### Spark on K8s 架构
我们离线任务大部分都已经迁移到了spark,spark任务运行在k8s有2种方式:
spark submit on k8s 和 spark operator on k8s。

- spark submit on k8s架构图
![spark submit on k8s架构图](../images/sparkOnK8s2.png)

- spark operator on k8s架构图
![spark operator on k8s架构图](../images/sparkOnK8s3.png)

以spark-submit方式提交到K8s集群是Spark在2.3版本后提供的原生功能，客户端通过spark-submit设置K8s的相关参数，
内部再调用K8sApi在K8s集群中创建Driver Pod，Driver再调用K8sApi创建需要的Executor Pod，共同组成Spark Application，
作业结束后Executor Pod会被Driver Pod销毁，而Driver Pod则继续存在直到被清理。使用spark-submit方式的最大好处是由spark-submit来与K8s的进行交换，
提交作业的方式几乎保持一致。但是因为使用的便利性所需要的封装也会带来一些缺点，spark-submit是通过K8sApi创建Pod，使用非声明式的提交接口，
如果需要修改K8s配置就需要重新开发新接口，二次开发复杂繁琐，虽然Spark提供了大量的K8s配置参数，但也远比不了K8s YAML的声明式的提交方式更加灵活，
而且Spark Application和K8s Workload的生命周期还不能较好地对应起来，生命周期不能灵活控制，任务监控也比较难接入Prometheus集群监控。
虽然Spark社区也不断地在推出新特性来和K8s集成地更加灵活，不过对于些复杂场景需要定制开发，spark-submit的封装性也会成为阻碍。

spark-submit还是离线任务提交的思维，而Spark Operator方式就更倾向于K8s作业的思维，作为K8s的自定义控制器，
在集成了原生的Spark on K8s的基础上利用K8s原生能力提供了更全面管控功能。Spark Operator使用声明式的YAML提交Spark作业，
并提供额外组件来管理Spark作业的生命周期，SparkApplication控制器，负责SparkApplicationObject的创建、更新和删除，同时处理各种事件和作业状态，
Submission Runner, 负责调用spark-submit提交Spark作业，Driver和Executor的运行流程是一致的，Spark Pod Monitor，负责监控和同步Spark作业相关Pod的状态。
Spark Operator最大的好处是为在K8s中的Spark作业提供了更好的控制、管理和监控的功能，
可以更加紧密地与K8s结合并能灵活使用K8s各种特性来满足复杂场景，例如混部场景，而相对地它也不再像spark-submit那样方便地提交任务，
所以如何使用Spark Operator优雅提交任务将是在离线混部中一项重要的工作。

最终我们还是选择了spark sumbit on k8s的方式，因为整个离线调度系统通过接入kyubbi统一了任务提交的方式，sumbit方式更符合我们现在的管理模式。
而且我们通过配置管理系统可以动态的将k8s的参数弄到pod模板，将spark的参数统一到配置中心管理，所以对于我们来说这种模式，更符合我们整个系统生态。

所以我们整体链路图如下:
![mars调度离线任务整体链路](../images/sparkOnK8s4.png)

### 2.2 技术组件栈
- **计算引擎**: Apache Spark 3.2.2
- **SQL网关**: Kyuubi 1.7.1
- **批调度器**: YuniKorn 1.3.0(替代Yarn)
- **Shuffle服务**: Celeborn 0.5.4 (替代ESS)
- **容器编排**: Kubernetes v1.23.2

## 三、核心技术模块详解

### 3.1 spark相关优化
#### 3.1.1 spark镜像
Spark任务容器化的第一步就是构建具有Spark相关环境的镜像,Spark任务类型主要分为sql任务和jar任务,
在实践的过程中我们发现Spark的镜像构建需要注意几个问题：

- Spark环境的完整性:镜像中除了打入自研的Spark包以外,还需要打入相应的依赖如Hadoop、ZSTD、RSS、paimon等包,
对于SparkJar任务还有直接调用Hadoop客户端的，因此Hadoop客户端也需要打入镜像中。

- 环境变量问题:镜像生成容器后需要预置如Spark、Hadoop的环境变量，如果镜像中相关目录的位置不能完全和Yarn的提交节点保持一致，
则需要检查各启动脚本，如spark-env.sh中的环境变量的路径是否存在，发生冲突时可以修改为绝对路径。

- spark日志:在后面的日志方案中会详细讲,我们在spark镜像中支持了日志上传hdfs功能。

#### 3.2.2 spark改造
Spark任务运行在K8s上，对于一些使用的兼容问题也进行了相关改造。

- HistoryServer改造，因为Spark on k8s没有存储已结束作业的日志，因此参考了on Yarn的方式，
在Spark作业结束后，通过日志上传脚本把Driver和Executor的日志上传HDFS，与Yarn日志聚合类似，同时也在Spark HistoryServer做了二次开发工作，
增加了on K8s方式的日志查看接口，用户查看已完成的Executor日志时，不再请求JobHistory Server，而是请求Spark HistoryServer接口。
但日志上传方式需要Executor执行完才能查看到日志，为了能实时查看到执行中的日志，我们实现了一个http服务用来请求k8s接口获取日志

- 主机ip通信，Spark Driver和Executor之间的通信通常是通过主机名进行的，然后通过executor通过链接driver service ip进行通信，当spark任务
变多dns和service频繁创建，解析，会造成极大压力所以我们直接去除了对应的K8s Service，通过访问driver ip而不是域名的方式来规避这个问题。

- 因为我们使用的spark是3.2.2版本，还对schedueler name配置进行适配可以让driver和executor使用不同的调度器调度，新版本已经支持了。

- spark在创建完driver后还会通过k8s-api获取自身信息，但有时会访问api超时，所以在spark内部做了超时重试机制。

- Spark任务的离线日志单独存储(和离线任务盘放到一起)，避免对在线业务pod的影响和磁盘负载高等问题。

### 3.2 混部k8s集群相关优化
随着spark任务变多,K8s集群压力变得很大，因为我们离线任务有个特点，除了凌晨运行的任务多之外,因为我们的计费规则
9点以后会比较便宜，然后大量任务会在9点运行，这部分都是那种P3的任务，而spark on k8s就是从p3开始迁移，所以当时整个
集群负载非常高:

cpu:
![k8s cpu](../images/sparkOnK8s5.png)

io:
![k8s io](../images/sparkOnK8s6.png)

集群pod运行数:
![集群pod运行数](../images/sparkOnK8s8.png)

后来我们对k8s进行优化主要几个方面:
- 拆分spark请求与在线业务请求api-server
- 将event写入独立etcd集群，保障正常业务数据写入稳定性
- 优化k8s参数

优化后 k8s负载明显降低:
![k8s cpu](../images/sparkOnK8s11.png)

![k8s io](../images/sparkOnK8s7.png)

优化前k8s部署架构：

![k8s部署架构](../images/sparkOnK8s9.png)

优化后k8s部署架构：
![k8s部署架构](../images/sparkOnK8s10.png)

### 3.3 k8s离线调度器
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

我们调研了离线k8s调度器,发现yunikorn符合我们需求，而且也支持类似yarn中的多队列管理，我们对其进行了验证测试。
- driver运行结束后，资源不会自动回收
- driver异常后创建的executor不会资源回收
另外spark on k8s 还会创建configmap和secret,这些资源并不会统一清理干净
我们在系统中会定时清理这些无用资源。避免资源对象太多导致的问题。
我们之前就遇到过configmap资源太多，导致yunikorn watch超时导致启动失败的问题：

![yunikorn异常](../images/sparkOnK8s12.png)


### 3.4 混部资源隔离与限制
![混部资源隔离与限制](../images/sparkOnK8s13.png)

spark on k8s后,在线业务和离线业务都在k8s统一管理，公用一个kubepods cgroup组，
我们需要把离线任务和在线业务分开，给离线业务单独的限制，避免离线把整个节点资源占满。
我们参考腾讯caleus进行二次开发，通过标签将离线pod的cgroup全都挪到单独的offline cgroup下面，
结合我们之前做的在离线混部整体资源的控制，同时限制了yarn+k8s offline整体的资源管理。


### 3.5 日志采集方案
![混部资源隔离与限制](../images/sparkOnK8s14.png)

日志主要有2部分：一部分是运行中任务日志查看，一部分是任务运行完日志查看。

我们二次开发了spark-history，spark on k8s上的日志链接都会转发到我们开发的一个spark-log-proxy代理，
代理会自动识别这个任务的pod是否还在运行，如果在运行就会从k8s-api获取日志，如果运行结束了就会从hdfs拉日志。

我们在spark镜像中打入了hdfs客户端，并增加了执行结束后上传日志到hdfs的功能。


### 3.6 运维相关系统适配
#### 3.6.1 技术挑战
从Yarn ResourceManager切换到YuniKorn调度器后，原有的任务计费和资源统计机制需要重新设计：

**Yarn模式下的计费流程：**
- ResourceManager记录任务资源使用情况
- 通过Yarn History Server获取任务完成信息
- 基于ApplicationAttempt和Container信息计算资源消耗

**K8s模式下的适配方案：**
- YuniKorn调度器提供任务调度信息
- K8s Metrics API提供资源消耗数据
- 通过Pod生命周期事件计算任务时长

另外还有任务资源统计、任务画像等等，都对k8s进行了适配

## 四、总结与收益

**技术收益：**
1. **统一调度架构**：实现在离线作业统一调度，提升资源利用率10-15%
2. **运维效率提升**：单一调度器降低运维复杂度，减少故障响应时间67%
3. **技术架构升级**：从传统Hadoop生态升级到云原生K8s生态

**业务收益：**
1. **成本节约**：年化硬件和运维成本节约1000万元以上
2. **性能提升**：任务执行效率提升5-10%，支撑业务快速发展
3. **技术竞争力**：建立业界领先的大数据计算平台

通过Spark on K8s项目的成功实施，我们将建立起业界领先的云原生大数据计算平台，为公司数字化转型和业务发展提供强有力的技术支撑。
项目的成功不仅体现在技术指标的提升，更重要的是为公司未来的技术架构演进奠定坚实的基础。

## 五、未来规划
