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
