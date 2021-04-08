# 阿里云 Node Local DNS

来源：

```
helm repo add ali-incubator https://aliacs-app-catalog.oss-cn-hangzhou.aliyuncs.com/charts-incubator/
helm pull ali-incubator/ack-node-local-dns
```
--


# Node Local DNS
> 限制:
> - 如果集群为以下网络模式的，请确认 Terway 版本为 v1.0.10.301 或更新版本后安装：
>   - 基于 IPVLAN 的 Terway ENI 多 IP 网络模式
>   - Terway ENI 独占网络模式
> - 此应用需要您手动设置相关参数才能进行安装，请仔细阅读参数说明
> - 此应用不支持 ASK 集群，以及托管版、专有云版集群中部署的 ECI 类型的 Pod
> - Node Local DNS 不提供 hosts、rewrite 等插件能力，仅作为 CoreDNS 的透明缓存代理，如有需要，可在 CoreDNS 配置中修改
> - 如果集群使用了 PrivateZone 的，需要修改 CoreDNS 配置后启用，详见 PrivateZone 兼容说明

# 简介
Node Local DNS 通过在集群节点上作为 DaemonSet 运行 DNS 缓存代理来提高集群 DNS 性能。

本 Helm Chart 主要由一个 Admission Webhook Deployment 和一组 DNS 缓存 DaemonSet 组成：
- Admission Webhook Deployment 可以被用来拦截 Pod 创建的请求，自动注入使用 DNS 缓存的 Pod DNSConfig。
- DNS 缓存 DaemonSet 可以在每个节点上创建一个虚拟网络接口（默认监听于 169.254.20.10 IP，如需修改请工单咨询），配合 Pod 的 DNSConfig 和节点上的网络配置，Pod 内产生的 DNS 请求会被该 DaemonSet 服务所代理。DaemonSet 服务内同样基于 CoreDNS 提供服务，但其仅具有代理和缓存的功能，请勿启用额外的插件能力（如 hosts、rewrite 等）。

Node Local DNS 工作原理如图所示：

<img src="https://aliware-images.oss-cn-hangzhou.aliyuncs.com/ACK/image/localdns.png" style="width:800px;">

# 安装
建议您通过 ACK 应用目录控制台安装本应用。如需通过 Helm 部署此应用，可以参考如下命令：
```bash
$ helm install --name node-local-dns --set env.upstream_ip=172.23.0.20 incubator/ack-node-local-dns --namespace kube-system
```

安装后请确认当前集群各 ECS 节点均已经启动 DNS 缓存 DaemonSet 组件（即 node-local-dns Pod 数量与集群 ECS 节点数一致），如果有节点上有 Taints 的，需要您手动修改 kube-system 下 node-local-dns 这个 DaemonSet 的 Toleration。

# 参数说明
| 参数 | 描述 | 默认值 |
| --- | --- | --- |
| env.upstream_ip | DNS 缓存 DaemonSet 通过 upstream_ip 访问 CoreDNS，详见下方说明 | "" |
| env.cluster_domain | 集群主域名 | "cluster.local" |
| env.enable_log | DNS 缓存 DaemonSet 是否查询日志 | false |
| controller.enabled | 是否启用 Admission Webhook | true |
| controller.image | Admission Webhook Deployment 的镜像 URL | registry.cn-hangzhou.aliyuncs.com/acs/node-local-dns-admission-controller |
| controller.imageTag | Admission Webhook Deployment 的镜像 Tag | v1.0.2-8b46b2f-aliyun |
| controller.replicas | Admission Webhook Deployment 副本数 | 2 |
| nodelocaldns.image.repository | DNS 缓存 DaemonSet 的镜像 URL | registry.cn-hangzhou.aliyuncs.com/acs/k8s-dns-node-cache |
| nodelocaldns.image.tag | DNS 缓存 DaemonSet 的镜像 Tag | v1.15.13-6-7e6778ac |
| nodelocaldns.init_image.repository | DNS 缓存 DaemonSet Init Container 的镜像 URL | registry.cn-hangzhou.aliyuncs.com/acs/busybox |
| nodelocaldns.init_image.tag | DNS 缓存 DaemonSet Init Container 的镜像 Tag | v1.29.2 |

特别说明：
- 本应用会创建一个名为 node-local-upstream，IP 地址为 `env.upstream_ip` 的 ClusterIP Service，用于 DNS 缓存 DamonSet 与 CoreDNS 进行通信；`env.upstream_ip` 需要指定为集群 Service 网段中尚未被占用的任意 IP。

# 使用方式

完成安装后，为了能使应用原本请求 CoreDNS 的流量改为由 DNS 缓存 DaemonSet 代理，需要使 Pod 内部的 /etc/resolv.conf 中 nameservers 配置成 `169.254.20.10` 和 `kube-dns` 对应的 IP 地址，您有以下几种方式可以选择：

- （推荐）借助 Admission Webhook Deployment 在 Pod 创建时做 DNSConfig 配置自动注入
- 创建 Pod 时手动指定 DNSConfig
- （不推荐，变更风险高）修改 kubelet 参数，并重启节点 kubelet

### 方式一：配置 DNSConfig 自动注入
Admission Webhook Deployment 可用于自动注入 DNSConfig 至新建的 Pod 中，避免您手工配置 Pod YAML 进行注入。
本应用默认会监听包含 `node-local-dns-injection=enabled` Label 标签的命名空间中新建 Pod 的请求，您可以通过以下命令给命名空间打上 Label 标签：

```bash
kubectl label namespace default node-local-dns-injection=enabled
```

开启自动注入后，您创建的 Pod 会被增加以下字段，为了最大程度上保证业务 DNS 请求高可用，nameservers 中会额外加入 `kube-dns` 的 ClusterIP 地址作为备份的 DNS 服务器：
```yaml
  dnsConfig:
    nameservers:
    - 169.254.20.10
    - 172.21.0.10
    options:
    - name: ndots
      value: "3"
    - name: attempts
      value: "2"
    - name: timeout
      value: "1"
    searches:
    - default.svc.cluster.local
    - svc.cluster.local
    - cluster.local
  dnsPolicy: None
```

在命名空间 DNSConfig 自动注入开启的情况下，如需对部分 Pod 进行豁免（即不进行注入），可以调整其 Pod Template 中 Labels 标签字段，加上 `node-local-dns-injection=disabled` 标签。

##### 未注入 DNSConfig 问题排查
自动注入需满足以下条件，请检查是否满足：
- 新建 Pod 不位于 `kube-system` 和 `kube-public` 命名空间；
- 新建 Pod 所在命名空间的 Labels 标签包含 `node-local-dns-injection=enabled`；
- 新建 Pod 所在命名空间的 Labels 不包含 ECI Pod 相关标签，如 `virtual-node-affinity-injection`、`eci`、`alibabacloud.com/eci` 等；
- 新建 Pod 没有被打上 `eci`、`alibabacloud.com/eci` 等 ECI 相关 Label 标签，或打上禁用 DNS 注入 `node-local-dns-injection=disabled` Label 标签；
- 新建 Pod 为 HostNetwork 且 DNSPolicy 为 ClusterFirstWithHostNet，或 Pod 为非 HostNetwork 且 DNSPolicy 为 ClusterFirst；

### 方式二：手动指定 DNSConfig
如果不适用 Admission Webhook Deployment 自动注入，可以参考以下方式进行手动指定 DNSConfig，请注意：
- dnsPolicy 必须为 None
- nameservers 配置成 `169.254.20.10` 和 `kube-dns` 的 ClusterIP 对应的 IP 地址
- 配置 searches，保证集群内部域名能够被正常解析
- ndots 默认为 5，可以适当降低 ndots 以提升解析效率，详见 [https://linux.die.net/man/5/resolv.conf](https://linux.die.net/man/5/resolv.conf)

```bash
apiVersion: v1
kind: Pod
metadata:
  name: alpine
  namespace: default
spec:
  containers:
  - image: alpine
    command:
      - sleep
      - "10000"
    imagePullPolicy: Always
    name: alpine
  dnsPolicy: None
  dnsConfig:
    nameservers: ["169.254.20.10","172.21.0.10"]
    searches:
    - default.svc.cluster.local
    - svc.cluster.local
    - cluster.local
    options:
    - name: ndots
      value: "3"
    - name: attempts
      value: "2"
    - name: timeout 
      value: "1"
```

### 方式三：配置 kubelet 启动参数
kubelet 通过 --cluster-dns 和 --cluster-domain 两个参数来全局控制 Pod DNSConfig。可以在 /etc/systemd/system/kubelet.service.d/10-kubeadm.conf 配置文件中修改其参数的配置，修改后 systemctl daemon-reload 和 systemctl restart kubelet 来生效。

```bash
--cluster-dns 169.254.20.10 --cluster-dns 172.21.0.10 --cluster-domain cluster.local
```

# 卸载
建议您通过 ACK 集群控制台卸载本应用。

```bash
$ helm delete --purge node-local-dns
```

# 存量升级
已经安装有本 Helm Chart 的 Kubernetes 集群如需进行升级，需要对已有 Helm Release 进行卸载后全新安装。 卸载前请进入应用 - Helm - 发布列表 - 详情 - 参数中确认此前安装的参数，并参考以下进行卸载：
- 如果此前安装参数中本地监听 IP 地址（local_dns_ip）为等同于 kube-dns 的 ClusterIP 地址，可以直接卸载
- 如果此前安装参数中的高可用开关（high_availability）为打开状态，本地监听 IP 地址（local_dns_ip）为 169.254.20.10，可以直接卸载
- 如果此前安装参数不满足以上条件，请按照以下流程进行卸载
    - 检查哪些命名空间中被打上了 node-local-dns-injection=enabled 的标签，并将标签删除，如 `kubectl label namespace default node-local-dns-injection-`
    - 检查这些命名空间中被注入 DNSConfig 的业务 Pod，并删除这些 Pod 以重建
    - 卸载

# PrivateZone 兼容性说明
node-local-dns 会默认采用 TCP 协议与 CoreDNS 进行通信，CoreDNS 会根据请求来源使用的协议与上游进行通信。
如果您集群中使用了 PrivateZone，经过 DNS 本地缓存组件的解析请求最终会以 TCP 协议请求至 PrivateZone 服务。
目前阿里云部分地域的 PrivateZone 服务尚未支持 TCP 协议，因此 PrivateZone 的解析结果可能会失败。
建议您使用以下方式，修改 CoreDNS 配置文件（命名空间 kube-system 下 coredns Configmap），在 forward 插件中指定请求上游的协议为 `perfer_udp`，修改之后 CoreDNS 会优先使用 UDP 协议与上游通信。
修改方式如下所示：
```yaml
# 修改前
forward . /etc/resolv.conf
# 修改后
forward . /etc/resolv.conf {
  prefer_udp
}
```
配置方式详见 https://coredns.io/plugins/forward/

# 其他
更多信息可以参考：
- DNS for Services and Pods [https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/#pod-dns-config](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/#pod-dns-config)
- Using NodeLocal DNSCache in Kubernetes clusters [https://kubernetes.io/docs/tasks/administer-cluster/nodelocaldns/](https://kubernetes.io/docs/tasks/administer-cluster/nodelocaldns/)