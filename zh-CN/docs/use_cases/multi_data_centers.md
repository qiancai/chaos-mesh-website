---
id: 多数据中心
title: 跨越多个数据中心的网络延迟模拟
sidebar_label: 跨越多个数据中心的网络延迟模拟
---

本文档帮助您模拟多个数据中心场景。

## 多个数据中心假设情景的特点

- 不同数据中心之间的延迟
- The bandwidth limitations between data centers

> **注意**:
> 
> 目前，Chaos Mesh无法模拟数据中心之间带宽限制的情况。 因此，在这种情况下，只模拟不同数据中心之间的延迟情况。

## Experiment environment

Suppose our application will be deployed in three data centers in a production environment and these data centers are still under construction. Now we want to test the impact of such a deployment topology on the business in advance.

Here we use TiDB cluster as an example. Suppose we already install the [TiDB cluster](https://docs.pingcap.com/tidb-in-kubernetes/stable/) and [Chaos Mesh](../user_guides/installation.md) in our Kubernetes environment. In this TiDB cluster, we have three TiDB pods, three PD pods and seven TiKV pods:

```bash
kubectl get pod -n tidb-cluster # "tidb-cluster" is the namespace of TiDB cluster
```

Output:

```bash
NAME                               READY   STATUS    RESTARTS   AGE
basic-discovery-7f9f48c465-6pdhn   1/1     Running   0          30m
basic-pd-0                         1/1     Running   0          30m
basic-pd-1                         1/1     Running   0          30m
basic-pd-2                         1/1     Running   0          30m
basic-tidb-0                       2/2     Running   0          29m
basic-tidb-1                       2/2     Running   0          29m
basic-tidb-2                       2/2     Running   0          29m
basic-tikv-0                       1/1     Running   0          29m
basic-tikv-1                       1/1     Running   0          29m
basic-tikv-2                       1/1     Running   0          29m
basic-tikv-3                       1/1     Running   0          29m
basic-tikv-4                       1/1     Running   0          29m
basic-tikv-5                       1/1     Running   0          29m
basic-tikv-6                       1/1     Running   0          29m
```

### Grouping

`dc-a`, `dc-b`, and `dc-c` are the three data centers we will use later. So we will split the pods to these data centers:

|      dc-a      |      dc-b      |       dc-c       |
|:--------------:|:--------------:|:----------------:|
|   basic-pd-0   |   basic-pd-1   |    basic-pd-2    |
|  basic-tidb-0  |  basic-tidb-1  |   basic-tidb-2   |
| basic-tikv-0/1 | basic-tikv-2/3 | basic-tikv-4/5/6 |

### Latency between three data centers

|                | latency |
|:--------------:|:-------:|
| dc-a <--> dc-b |   1ms   |
| db-a <--> dc-c |   2ms   |
| dc-b <--> dc-c |   2ms   |

## 注入网络延迟

### 设计注入规则

Chaos Mesh 提供 [`NetworkChaos`](../chaos_experiments/network_chaos. md) 注入网络延迟。 这样我们就可以用它来模拟三个数据中心之间的延迟。

目前， `NetworkChaos` 的限制是，每个目标点只有一个 `网上配置` 有效。 So we can use the following rules:

| source pods | latency | target pods |
|:-----------:|:-------:|:-----------:|
|    dc-a     |   1ms   |    dc-b     |
|    dc-a     |   1ms   |    dc-c     |
|    dc-b     |   1ms   |    dc-c     |
|    dc-c     |   1ms   |    dc-a     |
|    dc-c     |   1ms   |    dc-b     |

根据以上规则， `dc-a` 和 `dc-b` 之间的延迟是 `1ms` `dc-a` 和 `dc-c` 之间的延迟为 `2ms` 与 `dc-b` 和 `dc-c` 之间的延迟为 `2ms`。

### 定义chaos expert

根据注入规则，我们将混乱试验定义为：

```yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: NetworkChaos
metadata:
  name: network-delay-a
  namespace: tidb-cluster
spec:
  action: delay # chaos action
  mode: all
  selector: # define the pods belong to dc-a
    pods:
      tidb-cluster: # namespace of the target pods
        - basic-tidb-0
        - basic-pd-0
        - basic-tikv-0
        - basic-tikv-1
  delay:
    latency: '1ms'
  direction: to
  target:
    selector: # define the pods belong to dc-b and dc-c
      pods:
        tidb-cluster: # namespace of the target pods
          - basic-tidb-1
          - basic-tidb-2
          - basic-pd-1
          - basic-pd-2
          - basic-tikv-2
          - basic-tikv-3
          - basic-tikv-4
          - basic-tikv-5
          - basic-tikv-6
    mode: all

---
apiVersion: chaos-mesh.org/v1alpha1
kind: NetworkChaos
metadata:
  name: network-delay-b
  namespace: tidb-cluster
spec:
  action: delay
  mode: all
  selector: # define the pods belong to dc-b
    pods:
      tidb-cluster: # namespace of the target pods
        - basic-tidb-1
        - basic-pd-1
        - basic-tikv-2
        - basic-tikv-3
  delay:
    latency: '1ms'
  direction: to
  target:
    selector: # define the pods belong to dc-c
      pods:
        tidb-cluster: # namespace of the target pods
          - basic-tidb-2
          - basic-pd-2
          - basic-tikv-4
          - basic-tikv-5
          - basic-tikv-6
    mode: all

---
apiVersion: chaos-mesh.org/v1alpha1
kind: NetworkChaos
metadata:
  name: network-delay-c
  namespace: tidb-cluster
spec:
  action: delay
  mode: all
  selector: # define the pods belong to dc-c
    pods:
      tidb-cluster: # namespace of the target pods
        - basic-tidb-2
        - basic-pd-2
        - basic-tikv-4
        - basic-tikv-5
        - basic-tikv-6
  delay:
    latency: '1ms'
  direction: to
  target:
    selector: # define the pods belong to dc-a and dc-b
      pods:
        tidb-cluster: # namespace of the target pods
          - basic-tidb-0
          - basic-tidb-1
          - basic-pd-0
          - basic-pd-1
          - basic-tikv-0
          - basic-tikv-1
          - basic-tikv-2
          - basic-tikv-3
    mode: all
```

### 应用chaos测试

将上述chaos实验定义为 `延迟.yaml` 并应用此文件：

```bash
kubectl apply -f delay.yaml
```

### Check the result

Use `ping` command to check the latency between three centers.

#### Check the latency between the pods belong to `dc-a`

```bash
kubectl exec -it -n tidb-cluster basic-tidb-0 -c tidb -- ping -c 2 basic-tikv-0.basic-tikv-peer.tidb-cluster.svc
```

output:

```bash
PING basic-tikv-0.basic-tikv-peer.tidb-cluster.svc (10.244.1.229): 56 data bytes
64 bytes from 10.244.1.229: seq=0 ttl=63 time=0.095 ms
64 bytes from 10.244.1.229: seq=1 ttl=63 time=0.100 ms
```

From the output, we can see that the latency between the pods belong to `dc-a` is around `0.1ms`.

#### Check the latency between `dc-a` and `dc-c`

```bash
kubectl exec -it -n tidb-cluster basic-tidb-0 -c tidb -- ping -c 2 basic-tidb-1.basic-tidb-peer.tidb-cluster.svc
```

output:

```bash
PING basic-tidb-1.basic-tidb-peer.tidb-cluster.svc (10.244.3.3): 56 data bytes
64 bytes from 10.244.3.3: seq=0 ttl=62 time=1.193 ms
64 bytes from 10.244.3.3: seq=1 ttl=62 time=1.201 ms
```

从输出中我们可以看到 `dc-a` and `dc-c` 之间的延迟约为 `1ms`。

#### 检查 `dc-b` 和 `dc-c 之间的延迟`

```bash
kubectl exec -it -n tidb-cluster basic-tidb-0 -c tidb -- ping -c 2 basic-tidb-2.basic-tidb-peer.tidb-cluster.svc
```

输出：

```bash
PING basic-tidb-2.basic-tidb-peer.tidb-cluster.svc (10.244.2.27): 56 data bytes
64 bytes from 10.244.2.27: seq=0 ttl=62 time=2.200 ms
64 bytes from 10.244.2.27: seq=1 ttl=62 time=2.251 ms
```

从输出中我们可以看到 `dc-a` and `dc-c` 之间的延迟约为 `2ms`。

## 删除网络延迟

```bash
kubectl delete -f delay.yaml
```
