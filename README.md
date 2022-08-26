# hapv


# prom 安装

```bash
# 安装
kubectl apply --server-side -f kube-prom/setup

kubectl apply -f kube-prom/

# 删除
kubectl delete --ignore-not-found=true -f kube-prom/ -f kube-prom/setup

```

## prometheus-adapter

todo 配置编写流程？

you also need to configure custom rules
(example configuration https://github.com/DirectXMan12/k8s-prometheus-adapter/blob/master/docs/sample-config.yaml), 
otherwise prometheus-adapter will not serve the custom metrics API.

否则出现 404

```bash
status:
  conditions:
  - lastTransitionTime: "2022-08-24T03:20:24Z"
    message: 'failing or missing response from https://22.0.2.25:6443/apis/custom.metrics.k8s.io/v1beta1:
      bad status from https://22.0.2.25:6443/apis/custom.metrics.k8s.io/v1beta1: 404'
    reason: FailedDiscoveryCheck
```

添加规则以后，就会有相应的路由响应 serve 相关api

```bash
k get apiservices.apiregistration.k8s.io | grep metrics
v1beta1.external.metrics.k8s.io        monitoring/prometheus-adapter   True        8m56s
v1beta1.metrics.k8s.io                 monitoring/prometheus-adapter   True        8m56s
v1beta2.custom.metrics.k8s.io          monitoring/prometheus-adapter   True        8m56s

# api测试
kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta2/" | jq .
kubectl get --raw "/apis/metrics.k8s.io/v1beta1/" | jq .
kubectl get --raw "/apis/external.metrics.k8s.io/v1beta1/" | jq .


```

# 服务启动

通过prome采集数据以后

```bash
http_requests_total{endpoint="http", instance="22.0.0.16:3000", job="httpserver", namespace="monitoring", pod="httpserver-64db7f5658-d2pl2", service="httpserver", status="200"}


kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta2/namespaces/httpserver/pods/*/httpserver_requests_qps"

```

# 数据流向 

服务metrics接口采集数据-prom采集-prometheus-adapter积极性数据整合-hpa使用整合的数据-副本变更

# prometheus-adapter配置

https://github.com/kubernetes-sigs/prometheus-adapter/blob/master/docs/config.md

- adapter作用： 对metrics进行筛选，在hpa之前进行预处理 The adapter determines which metrics to expose, and how to expose them, through a set of "discovery" rules.

四步走
- 那些Prometheus的指标准备让adapter处理
- 这些指标所关联的k8s资源类型是啥
- 对这些筛选过的指标进行命名，使其可以通过custom metrics API查询得到
- 通过prom查询的具体语句

## 数据查询的关键

- 要想向prom查询想要的hpa相关的指标需要下面的关键信息
  - 指标名称是啥
  - 究竟要查哪些资源的相关的这个指标，那个ns的哪个pod
  - 要查多久之内的数据，时间间隔

```bash
kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta2/" | jq .

{
  "kind": "APIResourceList",
  "apiVersion": "v1",
  "groupVersion": "custom.metrics.k8s.io/v1beta2",
  "resources": [
    {
      "name": "namespaces/http_requests_qps",
      "singularName": "",
      "namespaced": false,
      "kind": "MetricValueList",
      "verbs": [
        "get"
      ]
    },
    {
      "name": "pods/http_requests_qps",
      "singularName": "",
      "namespaced": true,
      "kind": "MetricValueList",
      "verbs": [
        "get"
      ]
    }
  ]
}


# 查询具体的指标
kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta2/namespaces/monitoring/pods/*/http_requests_qps" | jq .

{
  "kind": "MetricValueList",
  "apiVersion": "custom.metrics.k8s.io/v1beta2",
  "metadata": {
    "selfLink": "/apis/custom.metrics.k8s.io/v1beta2/namespaces/monitoring/pods/%2A/http_requests_qps"
  },
  "items": [
    {
      "describedObject": {
        "kind": "Pod",
        "namespace": "monitoring",
        "name": "httpserver-64db7f5658-d2pl2",
        "apiVersion": "/v1"
      },
      "metric": {
        "name": "http_requests_qps",
        "selector": null
      },
      "timestamp": "2022-08-24T08:10:32Z",
      # 这里的 value: 200m，值的后缀“m” 标识 milli-requests per seconds，所以这里的 100m 的意思是 0.1/s 每秒0.1 个请求。
      "value": "200m"
    }
  ]
}
```

## 无法缩放
AbleToScale  False   FailedGetScale  the HPA controller was unable to get the target's current scale: deployments/scale.apps "sample-httpserver" not found

下面的字段要填写正确

```bash
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: httpserver
```
## hpa状态

```bash
Conditions:
  Type            Status  Reason              Message
  ----            ------  ------              -------
  AbleToScale     True    ReadyForNewScale    recommended size matches current size
  ScalingActive   True    ValidMetricFound    the HPA was able to successfully calculate a replica count from pods metric http_requests_qps
  ScalingLimited  False   DesiredWithinRange  the desired count is within the acceptable range
```

首先，AbleToScale 表明 HPA 是否可以获取和更新扩缩信息，以及是否存在阻止扩缩的各种回退条件。 
其次，ScalingActive 表明 HPA 是否被启用（即目标的副本数量不为零） 以及是否能够完成扩缩计算。 当这一状态为 False 时，通常表明获取度量指标存在问题。 
最后一个条件 ScalingLimited 表明所需扩缩的值被 HorizontalPodAutoscaler 所定义的最大或者最小值所限制（即已经达到最大或者最小扩缩值）。 这通常表明你可能需要调整 HorizontalPodAutoscaler 所定义的最大或者最小副本数量的限制了。

## 测试hpa

用到cpu使用率作为缩放指标的时候，容器的资源限制需要写上，cpu的hpa从<unknown>到显示指标需要等待时间长一些三到五分钟

```yaml
          resources:
            limits:
              cpu: 500m
            requests:
              cpu: 200m
```

```bash

k port-forward httpserver-64db7f5658-d2pl2 3000:3000

https://github.com/tsenart/vegeta/releases

# 加负载
while sleep 0.01; do wget -q -O- http://php-apache; done

# 压测结果
echo "GET http://127.0.0.1:3000" | vegeta attack -duration 60s -connections 10 -rate 240/s | vegeta report

Requests      [total, rate, throughput]         14400, 240.02, 239.08
Duration      [total, attack, wait]             1m0s, 59.996s, 234.553ms
Latencies     [min, mean, 50, 90, 95, 99, max]  44.962ms, 155.317ms, 85.478ms, 369.544ms, 497.953ms, 670.624ms, 844.616ms
Bytes In      [total, mean]                     172800, 12.00
Bytes Out     [total, mean]                     0, 0.00
Success       [ratio]                           100.00%
Status Codes  [code:count]                      200:14400
Error Set:

# hpa副本变化
k get hpa sample-httpserver -w
NAME                REFERENCE               TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
sample-httpserver   Deployment/httpserver   200m/50   1         5         1          18m
sample-httpserver   Deployment/httpserver   280m/50   1         5         1          19m
sample-httpserver   Deployment/httpserver   200m/50   1         5         1          19m
sample-httpserver   Deployment/httpserver   41200m/50   1         5         1          21m
sample-httpserver   Deployment/httpserver   184800m/50   1         5         1          22m
sample-httpserver   Deployment/httpserver   239239m/50   1         5         2          22m
sample-httpserver   Deployment/httpserver   121275m/50   1         5         4          22m
sample-httpserver   Deployment/httpserver   49928m/50    1         5         5          22m
sample-httpserver   Deployment/httpserver   11203m/50    1         5         5          23m
sample-httpserver   Deployment/httpserver   200m/50      1         5         5          23m

# hpa事件
3m15s       Normal    SuccessfulRescale        horizontalpodautoscaler/sample-httpserver   New size: 2; reason: pods metric http_requests_qps above target
3m          Normal    SuccessfulRescale        horizontalpodautoscaler/sample-httpserver   New size: 4; reason: pods metric http_requests_qps above target
2m45s       Normal    SuccessfulRescale        horizontalpodautoscaler/sample-httpserver   New size: 5; reason: pods metric http_requests_qps above target
```

# hpa

字段介绍
https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.24/#horizontalpodautoscalerspec-v2-autoscaling
 主要metrics和behavior


## 创建hpa命令行

kubectl autoscale deployment httpserver --cpu-percent=10 --min=1 --max=3


## metrics

https://kubernetes.io/zh-cn/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/#autoscaling-on-multiple-metrics-and-custom-metrics


- 类型分为  "ContainerResource", "External", "Object", "Pods" or "Resource"
- resource 就是pod资源中的resource字段的类型
- pods 和 Object 是自定义的指标 custom metrics
- pods 这些指标从某一方面描述了 Pod， 在不同 Pod 之间进行平均，并通过与一个目标值比对来确定副本的数量。 它们的工作方式与资源度量指标非常相像，只是它们 仅 支持 target 类型为 AverageValue
- Object 这些度量指标用于描述在相同名字空间中的别的对象，而非 Pod。 请注意这些度量指标不一定来自某对象，它们仅用于描述这些对象。 对象度量指标支持的 target 类型包括 Value 和 AverageValue。 如果是 Value 类型，target 值将直接与 API 返回的度量指标比较， 而对于 AverageValue 类型，API 返回的度量值将按照 Pod 数量拆分， 然后再与 target 值比较
- 多个指标怎么办 如果你指定了多个上述类型的度量指标，HorizontalPodAutoscaler 将会依次考量各个指标。 HorizontalPodAutoscaler 将会计算每一个指标所提议的副本数量，然后最终选择一个最高值
- ContainerResource 容器资源指标的支持使得你可以为特定 Pod 中最重要的容器配置规模扩缩阈值。 例如，如果你有一个 Web 应用和一个执行日志操作的边车容器，你可以基于 Web 应用的 资源用量来执行扩缩，忽略边车容器的存在及其资源用量 容器资源指标 https://kubernetes.io/zh-cn/docs/tasks/run-application/horizontal-pod-autoscale/#container-resource-metrics
- External 与 Kubernetes 集群中的任何对象没有明显关系的度量指标进行自动扩缩， 例如那些描述与任何 Kubernetes 名字空间中的服务都无直接关联的度量指标。 在 Kubernetes 1.10 及之后版本中，你可以使用外部度量指标（external metrics）。https://kubernetes.io/zh-cn/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/#autoscaling-on-metrics-not-related-to-kubernetes-objects


## behavior
https://kubernetes.io/zh-cn/docs/tasks/run-application/horizontal-pod-autoscale/#scaling-policies
扩缩策略
可以在规约的 behavior 部分中指定一个或多个扩缩策略。当指定多个策略时， 允许最大更改量的策略是默认选择的策略。以下示例显示了缩小时的这种行为：

behavior:
scaleDown:
policies:
- type: Pods
value: 4
periodSeconds: 60
- type: Percent
value: 10
periodSeconds: 60
periodSeconds 表示在过去的多长时间内要求策略值为真。 第一个策略（Pods）允许在一分钟内最多缩容 4 个副本。第二个策略（Percent） 允许在一分钟内最多缩容当前副本个数的百分之十。

由于默认情况下会选择容许更大程度作出变更的策略，只有 Pod 副本数大于 40 时， 第二个策略才会被采用。如果副本数为 40 或者更少，则应用第一个策略。 例如，如果有 80 个副本，并且目标必须缩小到 10 个副本，那么在第一步中将减少 8 个副本。 在下一轮迭代中，当副本的数量为 72 时，10% 的 Pod 数为 7.2，但是这个数字向上取整为 8。 在 autoscaler 控制器的每个循环中，将根据当前副本的数量重新计算要更改的 Pod 数量。 当副本数量低于 40 时，应用第一个策略（Pods），一次减少 4 个副本。

可以指定扩缩方向的 selectPolicy 字段来更改策略选择。 通过设置 Min 的值，它将选择副本数变化最小的策略。 将该值设置为 Disabled 将完全禁用该方向的扩缩。

```go
// 策略只有两种
const (
// PodsScalingPolicy is a policy used to specify a change in absolute number of pods.
PodsScalingPolicy HPAScalingPolicyType = "Pods"
// PercentScalingPolicy is a policy used to specify a relative amount of change with respect to
// the current number of pods.
PercentScalingPolicy HPAScalingPolicyType = "Percent"
)


//策略选择有三种

// 选择最小和最大策略 或者禁用缩容 默认选择变化最大的变更策略
// ScalingPolicySelect is used to specify which policy should be used while scaling in a certain direction
type ScalingPolicySelect string

const (
// MaxPolicySelect selects the policy with the highest possible change.
MaxPolicySelect ScalingPolicySelect = "Max"
// MinPolicySelect selects the policy with the lowest possible change.
MinPolicySelect ScalingPolicySelect = "Min"
// DisabledPolicySelect disables the scaling in this direction.
DisabledPolicySelect ScalingPolicySelect = "Disabled"
)

```



# crane

## Crane install

```bash
k apply -f craned

k apply -f crane-crd

k apply -f crane-agent 
```

## Crane Metric-Adapter

Kubernetes 限制一个 ApiService 只能配置一个后端服务，因此，为了在一个集群内使用 Crane 提供的 Metric 和 PrometheusAdapter 提供的 Metric，Crane 支持了 RemoteAdapter 解决此问题

Crane Metric-Adapter 支持配置一个 Kubernetes Service 作为一个远程 Adapter
Crane Metric-Adapter 处理请求时会先检查是否是 Crane 提供的 Local Metric，如果不是，则转发给远程 Adapter

```bash
k get apiservices.apiregistration.k8s.io | grep metrics
v1beta1.custom.metrics.k8s.io          crane-system/metric-adapter     True        3m40s
v1beta1.external.metrics.k8s.io        crane-system/metric-adapter     True        27h
v1beta1.metrics.k8s.io                 monitoring/prometheus-adapter   True        27h

# metric-adapter 通过代理的方法将他自己不提供的metrics通过remote的方式转发到prometheus-adapter 
apiVersion: apps/v1
kind: Deployment
metadata:
  name: metric-adapter
  namespace: crane-system
spec:
  template:
    spec:
      containers:
      - args:
          #添加外部 Adapter 配置
        - --remote-adapter=true
        - --remote-adapter-service-namespace=crane-system
        - --remote-adapter-service-name=prometheus-adapter
        - --remote-adapter-service-port=443
```

## 创建epha

```bash
k apply -f ehpa.yaml


```

# 问题

- prom查询不到其他ns的资源
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: prometheus-k8s
  labels:
    app.kubernetes.io/component: prometheus
    app.kubernetes.io/instance: k8s
    app.kubernetes.io/name: prometheus
    app.kubernetes.io/part-of: kube-prometheus
    app.kubernetes.io/version: 2.36.1
rules:
  - verbs:
      - get
      - list
      # ts=2022-08-25T12:59:17.528Z caller=klog.go:116 level=error component=k8s_client_runtime func=ErrorDepth msg="pkg/mod/k8s.io/client-go@v0.24.0/tools/cache/reflector.go:167: Failed to watch *v1.Pod: unknown (get pods)"
      - watch
    apiGroups:
      - ''
    resources:
      - nodes/metrics
      - pods
      - services
      - endpoints
  - verbs:
      - get
    nonResourceURLs:
      - /metrics


```

# cpu 指标没有 

kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1/namespaces/crane-system/pods/*/crane_pod_cpu_usage" | jq .

# crd资源

```bash
analytics                           analytics    analysis.crane.io/v1alpha1             true         Analytics
configsets                          cs           analysis.crane.io/v1alpha1             true         ConfigSet
recommendationrules                 rr           analysis.crane.io/v1alpha1             false        RecommendationRule
recommendations                     recommend    analysis.crane.io/v1alpha1             true         Recommendation


effectivehorizontalpodautoscalers   ehpa         autoscaling.crane.io/v1alpha1          true         EffectiveHorizontalPodAutoscaler
effectiveverticalpodautoscalers     evpa         autoscaling.crane.io/v1alpha1          true         EffectiveVerticalPodAutoscaler
substitutes                         subs         autoscaling.crane.io/v1alpha1          true         Substitute


avoidanceactions                    avoid        ensurance.crane.io/v1alpha1            false        AvoidanceAction
nodeqosensurancepolicies            nep          ensurance.crane.io/v1alpha1            false        NodeQOSEnsurancePolicy
podqosensurancepolicies             qep          ensurance.crane.io/v1alpha1            true         PodQOSEnsurancePolicy


clusternodepredictions              cnp          prediction.crane.io/v1alpha1           true         ClusterNodePrediction
timeseriespredictions               tsp          prediction.crane.io/v1alpha1           true         TimeSeriesPrediction
```
##  推荐

通过 analytics.analysis.crane.io 创建后可以创建资源 recommendationrules.analysis.crane.io

recommendationrules.analysis.crane.io 创建 recommendations.analysis.crane.io 显示推荐的结果 资源或者副本

## ehpa

effectivehorizontalpodautoscalers.autoscaling.crane.io 创建 集群 hpa和timeseriespredictions.prediction.crane.io