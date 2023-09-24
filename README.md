# k8s-gzctf-deployment

gzctf k8s 部署工具 （实验室内部使用）

### 建立集群
#### `环境`
- `ubuntu 20.04`
- `最少两台物理机（或虚拟机），以双节点实例部署为例。`
	- `A节点`
		- `master主机`
		- `IP地址：192.168.123.80`
		- `hostname：k8s-master`
	- `B节点`
		- `worker主机`
		- `IP地址： 192.168.123.81`
		- `hostname： k8s-worker`
- `nfs 存储主机（可选`
	- `C存储`
		- `IP地址： 192.168.123.79`
#### `k8s部署工具安装（以1.25.0为例`
##### `建立环境`
- `替换hostname`
``` bash
hostnamectl set-hostname $hostname # $hostname为k8s-master或k8s-worker
sed -i -e "s/127.0.1.1.*/127.0.1.1 $hostname/g" /etc/hosts # 更新hosts文件
echo $ip_addr" "$hostname >> /etc/hosts # 同2
```
- `关闭swap并持久化DNS（虚拟机`
``` bash

swapoff -a # 当前关闭，重启会重开

sed -i -e "s/.*\(\/swap.img.*\)/#\1/g" /etc/fstab # 永久关闭swap

systemctl disable ufw --now # 关闭防火墙

apt install -y resolvconf # 安装resolvconf，持久化DNS

echo "nameserver 8.8.8.8" >> /etc/resolvconf/resolv.conf.d/head

systemctl enable --now resolvconf

resolvconf -u

```
##### `安装docker（已装跳过`
``` bash

apt-get install -y ca-certificates curl gnupg lsb-release

curl -fsSL http://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | sudo apt-key add -

add-apt-repository "deb [arch=amd64] http://mirrors.aliyun.com/docker-ce/linux/ubuntu $(lsb_release -cs) stable"

apt-get update

apt-get install -y docker-ce docker-ce-cli containerd.io

mkdir /etc/docker

touch /etc/docker/daemon.json

echo '{"registry-mirrors": ["https://docker.mirrors.ustc.edu.cn","https://hub-mirror.c.163.com","https://reg-mirror.qiniu.com","https://registry.docker-cn.com"],"exec-opts": ["native.cgroupdriver=systemd"]}' > /etc/docker/daemon.json

systemctl daemon-reload > /dev/null 2>&1

systemctl restart docker > /dev/null 2>&1

systemctl enable docker > /dev/null 2>&1

```

#####  `安装cri-docker，让k8s适配docker（1.25.0需要`
``` bash

apt install -y "./"$(ls | grep cri-dockerd) # cri-dockerd 由 https://github.com/Mirantis/cri-dockerd/releases 下载`

sed -i -e 's#ExecStart=.*#ExecStart=/usr/bin/cri-dockerd --pod-infra-container-image=registry.aliyuncs.com/google_containers/pause:3.8 --container-runtime-endpoint fd:// --network-plugin=cni --cni-bin-dir=/opt/cni/bin --cni-cache-dir=/var/lib/cni/cache --cni-conf-dir=/etc/cni/net.d#g' /usr/lib/systemd/system/cri-docker.service # 十分重要的一步，设置kubelet自动启动的pause容器版本，因为下文pull API对象容器镜像的时候，1.25.0版本k8s会自动获取pause:3.8，所以需要更改kubelet默认启动pause版本，不然会导致集群初始化完成不了。

systemctl daemon-reload # 配置使能

systemctl enable --now cri-docker # 使cri-docker服务开机自启

systemctl restart cri-docker # 重启cri-dockerd服务

sed -i -e "s/disabled_plugins\(.*\)/#disabled_plugins\1/g" /etc/containerd/config.toml # 注释掉containerd的配置，不然会导致cri-dockerd无法使用

```

##### `安装k8s集群配置工具`
``` bash

apt-get install -y apt-transport-https

# 添加对应key
curl https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | apt-key add -

#添加镜像
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
EOF

apt-get update

apt-get install -y kubelet=1.25.0-00 kubeadm=1.25.0-00 kubectl=1.25.0-00

apt-mark hold kubelet kubeadm kubectl

mkdir /etc/sysconfig

# 配置kubelet使用cri-dockerd.sock来管理容器
echo "KUBELET_KUBEADM_ARGS=\"--container-runtime=remote --container-runtime-endpoint=/run/cri-dockerd.sock\"" > /etc/sysconfig/kubelet

# 使kubelet服务开机自启
systemctl enable --now kubelet

```

##### `万事俱备，只欠东风（初始化集群`
``` bash

kubeadm config images pull --image-repository=registry.aliyuncs.com/google_containers --cri-socket unix:///run/cri-dockerd.sock # 拉取与当前k8s版本相对应的镜像

kubeadm init --control-plane-endpoint=$ip_addr --kubernetes-version=v1.25.0 --pod-network-cidr=10.244.0.0/16 --service-cidr=10.96.0.0/12 --token-ttl=0 --cri-socket=unix:///run/cri-dockerd.sock --upload-certs --image-repository registry.aliyuncs.com/google_containers # 初始化k8s集群，$ip_addr为k8s-master的ip，也就是192.168.123.80

export KUBECONFIG=/etc/kubernetes/admin.conf # 将对应config设为环境变量

mkdir -p $HOME/.kube # 创建用于存储config的目录

cp -i /etc/kubernetes/admin.conf $HOME/.kube/config # cpoy config到刚刚的目录

chown $(id -u):$(id -g) $HOME/.kube/config # 更改权限

kubectl apply -f ./kube-flannel.yml # 应用flannel cni网络，可从 https://github.com/flannel-io/flannel/releases 获取

```

`初始化master主机后，就可以初始化worker主机了，原理大差不差，后面会有我自己写的初始化脚本，在ubuntu20能完美运行。`

#### `k8s-1.25.0集群建立通用脚本（ubuntu 20.04可成功运行`：

`kubeadm、kubectl、kubelet等k8s集群部署工具`
`k8s集群一键部署工具（目前仅ubuntu20测试过：`
![[kube-deploy.tar]]
`master目录中的脚本用于部署master节点`
`worker目录中的脚本用于建立k8s基础环境，最后使用master节点建立成功后的join输出脚本即可加入集群之中
`
### `用于部署gzctf平台的yaml脚本`
`使用docker进行动态容器的构建`
`如下yaml脚本是我自己在自己的虚拟机进行测试使用的，也能成功部署gzctf，但是其架构是 1 个master主机，3 个worker主机，外加一台nfs存储主机`
`具体架构如下`
- `k8s-master`
	- `master主机`
	- `IP地址：192.168.227.80`
	- `hostname: k8s-master`
- `k8s-worker`
	- `worker主机`
	- `IP地址：192.168.227.81`
	- `hostname：k8s-worker`
	- `实例部署`
		- `postgres实例`
- `k8s-worker1`
	- `worker主机兼nfs存储`
	- `IP地址：192.168.227.82`
	- `hostname：k8s-worker1`
	- `实例部署`
		- `redis实例`
- `k8s-worker2`
	- `worker主机`
	- `IP地址：192.168.227.83`
	- `hostname：k8s-worker2`
	- `实例部署`
		- `gzctf平台`

**`对一些k8s中常见对象的解释（我替你向chatGPT问的qwq:`**
**`pod详解`**
`1. 容器的封装：Pod封装了一个或多个容器，它们通常会一起运行以构建一个应用程序的不同部分。这些容器可以在同一个Pod中协同工作，共享相同的资源，从而简化了它们之间的通信和协作。`
`2. 资源共享：Pod中的容器可以共享相同的网络和存储卷，这使得它们之间可以轻松地进行通信和数据共享。这对于需要多个容器协同工作的应用程序非常有用，例如，一个Web应用程序容器和一个辅助容器，用于日志收集或数据处理。`
`3. 单一调度单元：Pod是Kubernetes的调度单元，Kubernetes调度器会将整个Pod分配给某个节点上的工作。这确保了Pod中的所有容器在同一个节点上运行，以便它们可以直接通信。`
`4. 生命周期管理：Pod可以共享相同的生命周期。这意味着当Pod启动或停止时，其中的所有容器都将同时启动或停止。这有助于确保相关容器之间的一致性状态。`
`5. Sidecar容器：Pod中的一个常见用途是将主要应用程序容器与一个或多个Sidecar容器组合在一起。Sidecar容器可以执行附加任务，例如日志记录、监控、数据同步等，而不会影响主要应用程序容器。`
`6. 共享网络：Pod中的容器共享相同的网络命名空间，它们可以使用`localhost`来进行通信，这使得它们之间的通信更加高效。此外，它们可以通过同一IP地址和端口暴露服务。`
`7. 共享存储：Pod中的容器可以共享相同的存储卷，这对于需要在容器之间共享数据的应用程序非常有用。例如，你可以在Pod中运行一个数据库容器和一个数据备份容器，它们可以访问相同的存储卷来进行数据备份。`

**`service详解`**
`1. 稳定的网络访问：Service为一组具有相同标签选择器的Pod提供了一个稳定的网络入口点（Cluster IP），以便其他应用程序可以通过该IP地址访问这组Pod。无论Pod如何变化（例如，重新启动、扩展或缩减），Service的Cluster IP都会保持不变，因此客户端无需更改其连接信息。`
`2. 负载均衡：Service可以配置为在一组Pod之间分发入站流量，从而实现负载均衡。它通过Round Robin（轮询）或其他负载均衡算法将请求均匀地分发到多个Pod上，从而提高应用程序的可伸缩性和可用性。`
`3. 内部服务发现：在Kubernetes集群中，Service用于实现内部服务发现。其他Pod可以通过Service的DNS名称或Cluster IP来访问相关的服务，而无需了解具体的Pod名称或IP地址。这样，服务之间的通信变得更加简单和可维护。`
`4. 外部服务暴露：Service还可以用于将应用程序暴露给外部流量，例如通过NodePort或LoadBalancer类型的Service。这允许外部客户端通过Kubernetes节点的公共IP地址访问服务。LoadBalancer类型的Service还可以与云提供商的负载均衡器集成，以提供更高级的外部访问控制。`
`5. 头部选择器：Service可以配置为通过标签选择器选择要与之关联的Pod。这意味着你可以选择将特定标签的Pod关联到特定的Service，从而更精确地控制服务的成员。`
`6. 端口映射：Service允许你将外部端口映射到内部Pod的端口，这有助于将外部请求路由到正确的容器端口。这对于多容器Pod的应用程序非常有用，其中不同容器可能侦听不同的端口。`
`7. 健康检查和自愈：Service可以通过与Readiness Probe和Liveness Probe结合使用来监控关联的Pod的健康状态。如果某个Pod不健康，Service将不再将流量路由到该Pod，从而帮助确保应用程序的稳定性和可用性。`

**`deployment详解`**
`1. 自动副本管理：Deployment允许你定义一个或多个Pod副本的期望状态，然后它会自动处理Pod的创建、扩展、收缩和终止，以确保实际副本数与期望状态保持一致。这有助于实现应用程序的自动水平扩展。`
`2. 滚动更新：Deployment支持滚动更新策略，可以平滑地将应用程序从一个版本升级到另一个版本。通过逐步替换旧的Pod副本，Deployment确保了应用程序的可用性和稳定性。滚动更新还允许在更新过程中监控应用程序的健康状态，以便及时回滚到先前的稳定版本。`
`3. 回滚：如果升级后发生问题，Deployment允许你执行回滚操作，将应用程序恢复到之前的版本。这可以通过修改Deployment的配置来实现，Deployment将自动触发回滚操作并替换Pod副本。`
`4. 应用程序的版本控制：通过创建不同版本的Deployment，你可以轻松管理应用程序的多个版本。这对于在不同环境中测试和部署应用程序（如开发、测试和生产环境）非常有用。`
`5. 标签选择器：Deployment可以与标签选择器结合使用，以便选择要管理的Pod。这允许你在不影响整个应用程序的情况下更改或扩展特定组件。`
`6. 模板化配置：Deployment使用Pod模板来定义要部署的容器化应用程序的配置。这使得应用程序的配置可以在多个环境中重复使用，从而降低了部署的复杂性。`
`7. 应用程序自愈：Deployment支持Liveness Probe和Readiness Probe，这些探测器可用于监控Pod的健康状态。如果Pod不健康，Deployment可以自动重启或停止不健康的Pod，并确保应用程序保持可用。`

**`ingress详解`**
`1. HTTP和HTTPS路由：Ingress允许你定义HTTP和HTTPS请求的路由规则，以确定如何将请求路由到不同的服务或后端Pod。这使得你可以根据不同的域名、路径、请求头等条件来将流量导向不同的服务。`
`2. 主机名路由：你可以使用Ingress来将不同的域名映射到不同的后端服务。例如，你可以将`example.com`的流量路由到一个服务，将`api.example.com`的流量路由到另一个服务。`
`3. 路径路由：Ingress允许你根据请求的路径来路由流量。这使得你可以将流量根据路径分发到不同的服务，例如将`/app`的流量路由到一个前端应用程序服务，将`/api`的流量路由到一个API服务。`
`4. SSL/TLS终止：Ingress支持SSL/TLS终止，可以在Ingress规则中配置。这允许你在Ingress上终止SSL/TLS加密，然后将未加密的流量路由到后端服务。这对于保护外部流量的隐私和安全非常有用。`
`5. 负载均衡：Ingress可以配置在多个后端服务之间均衡流量。它支持轮询、IP哈希、会话粘滞等负载均衡算法，以提高应用程序的可伸缩性和性能。`
`6. 安全策略：Ingress可以配置安全策略，例如基于IP的访问控制、身份验证、授权等，以增强流量的安全性。这有助于防止未经授权的访问和保护应用程序免受攻击。`
`7. 自动发现：Ingress Controller可以自动发现并配置Ingress资源，这简化了应用程序的路由管理。你只需创建和维护Ingress资源，而Ingress Controller会自动处理路由规则和代理配置。`
`8. 灵活性：Ingress资源可以根据需求配置，从而允许你根据应用程序的特定要求定义复杂的路由策略。这使得Ingress非常灵活，可以适应各种不同的应用程序部署`

`可以使用 kubectl explan 指令获取某些对象的详细解释及用法介绍，例如存储卷之类的`

``` yaml
#! k8sconfig Secret 注：若容器启动使用的是docker，则无用
apiVersion: v1
kind: Secret
metadata:
  name: gzctf-k8sconfig
type: Opaque
stringData:
  k8sconfig: |
    apiVersion: v1
    clusters:
    - cluster:
        certificate-authority-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUMvakNDQWVhZ0F3SUJBZ0lCQURBTkJna3Foa2lHOXcwQkFRc0ZBREFWTVJNd0VRWURWUVFERXdwcmRXSmwKY201bGRHVnpNQjRYRFRJek1EZ3lOekU1TXpNMU9Gb1hEVE16TURneU5ERTVNek0xT0Zvd0ZURVRNQkVHQTFVRQpBeE1LYTNWaVpYSnVaWFJsY3pDQ0FTSXdEUVlKS29aSWh2Y05BUUVCQlFBRGdnRVBBRENDQVFvQ2dnRUJBTUpUClRidHkwRHBMbzZ6d01YZ21sMC9Jc3ZubmsxTWpsVEdIandpQWxVWU8yMFFFazVlejJPbDFaNzF4TE1meURmbFoKUEtaSEEwY0J2Tm5iL3E2UDgzM2xEa08zNWNHazJoR1MvYXI1WVdWSm5YQ2xGc3NMNmcxL0dDZzdELy9XaXpvNwp2VlA2R2FSSlVucE5kRWhJTmdRandoMTlZNnNxQ2ppcStJWE9Fdm4yanFoTXFCTXNlTjR5ZmppSkZYbTlndlNaCjFFNkgraFBTNWxBNjFHQXlYQ1pDWGdmY2VHZzBnbXE0TnZnWGM5VENaMDloKzFtMHNqc2RYVmVnTmRtUU8yQlIKRlRqL3B1eG50ejVoaWx4SmVUcFRPNU5tdnlhanlUT2JzOGM3RGIvN0tja3Y3RUdmR0lKWGtIMjAzdCtOcjBHLwpZd3lMNStHSHViTU9DNnJPTVk4Q0F3RUFBYU5aTUZjd0RnWURWUjBQQVFIL0JBUURBZ0trTUE4R0ExVWRFd0VCCi93UUZNQU1CQWY4d0hRWURWUjBPQkJZRUZQTGJXbXhOQnFacGRFZmlTWkdYRGhrQ2pySlRNQlVHQTFVZEVRUU8KTUF5Q0NtdDFZbVZ5Ym1WMFpYTXdEUVlKS29aSWh2Y05BUUVMQlFBRGdnRUJBRXRqZzdIcDljKzdqME9YS2pEUgpQdk5Mek9DUm9idmNRbFVNZ001dlo1S2hvaDJDSW5kbWJSQ0tGQ0hIa1ZmUEMyR1B0K0R0WGRMb2dZZnpxZ2ZiCjFtcm05VG5rOXArU3VmMDkvOWNYUmNtMUN2RlRQUnVreHlmdlNnRmZncVc4RkxCR1B5QmpodGFWblZVZTZrRkcKVXlKV3hlYm9VcjY3MGgxVnEvUE54bERCck1RcTlYS2JndGlkUTM3dmxwQlJ0QS83VGFzZm42RnF5a1V0cU9DbgpTcnZHQ05aRW82bnlwb1E0WnBuYmFpTUVrZVE4QncrS21lUXlKcTRUMlBGZFlNcS9WRUgvM2Q5MytCTFpwdHlvClhpbFVnbWE2bG1PNHRmOGExek5PVjN5VXhid3JlQ3I5SGt2SDd2M09obUpFQ25sV1NCKzFLR1gxY2FxaXZ5RmEKbE1BPQotLS0tLUVORCBDRVJUSUZJQ0FURS0tLS0tCg==
        server: https://192.168.227.129:6443
      name: kubernetes
    contexts:
    - context:
        cluster: kubernetes
        user: kubernetes-admin
      name: kubernetes-admin@kubernetes
    current-context: kubernetes-admin@kubernetes
    kind: Config
    preferences: {}
    users:
    - name: kubernetes-admin
      user:
        client-certificate-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSURJVENDQWdtZ0F3SUJBZ0lJVXAvYzV5OXRuNkV3RFFZSktvWklodmNOQVFFTEJRQXdGVEVUTUJFR0ExVUUKQXhNS2EzVmlaWEp1WlhSbGN6QWVGdzB5TXpBNE1qY3hPVE16TlRoYUZ3MHlOREE0TWpZeE9UTTBNREZhTURReApGekFWQmdOVkJBb1REbk41YzNSbGJUcHRZWE4wWlhKek1Sa3dGd1lEVlFRREV4QnJkV0psY201bGRHVnpMV0ZrCmJXbHVNSUlCSWpBTkJna3Foa2lHOXcwQkFRRUZBQU9DQVE4QU1JSUJDZ0tDQVFFQXlVNjU3dE1UaVRxQnhZOWUKWmpPRk1ZREo1cVJUTXphTFN6YnVuS205RTJzNlNXd1oyNkk1bE01VzFtQXJLS1A5Y3NEWlBNWU1wQ2IraVRmegozT0U4ZjNzZzQzbWhIaDBRcWRGbzF0Vk9yZlgyRk1kT0NwMkZBYXpZTEV1VWdRUlNDK0pZV2JPZHppd2hxTVB5Ckk0KzFUTmV2VitQVHA0ZWJZR1RqTEpOYzBZNUhaTEg0eG01NVNlSHNoSUFOOVNrVFdONkFRaHhaVVlDS1pSQnIKdTkzRmNNV2ZzcU1JYzUwYjBxQ0JkL0d6U2lUS0NEWFJtUHdRMVhNcW5TS3ptTHY5ME5ET1hiNnVIeGJLbWJYbApUc1B6VDZDTnlURFBOdWQrb1hYZElsR2haaWVBMi9zbnIrWEw0bk82UlA0eHBUMnlJSkFEbXR0YkQzZ1dzSHVNClZRYW84d0lEQVFBQm8xWXdWREFPQmdOVkhROEJBZjhFQkFNQ0JhQXdFd1lEVlIwbEJBd3dDZ1lJS3dZQkJRVUgKQXdJd0RBWURWUjBUQVFIL0JBSXdBREFmQmdOVkhTTUVHREFXZ0JUeTIxcHNUUWFtYVhSSDRrbVJsdzRaQW82eQpVekFOQmdrcWhraUc5dzBCQVFzRkFBT0NBUUVBUTVZVWxFalliS3RhcW1IODRDN3lWS3liYmhncGJ4d0JyVjRSCkF2VWcyZXVDS1NBNjQ5WmE4MWtjMzY1UVRZNGQ0L2F0SHljd2dQR1JIUHVPSkV4d2w2QnMvYkREemk4aW5HSkMKTTdjOFRUSmxxZnF1bzZ0MmdZV2Q3VW9oeDZnNC9OVzBuRVpoSHZkWFRTMFh1dmxwc3pKY2xkdldkY2VZWXAxTApDMDFDbmJkWjVMeU1BL1dLSjZNRDZrei9YSEkzbjZDTGRvcTlvRmN3K0RnMXJ2UTZoMDRiREdtdC9IWlpoSnJPCjJyQzdVeUZLR3N6eGY1S1B6elU5RUdaTU0zaGk5UXJlS1hWQVU4RUVSU3hTSlF3VVQxOEVtM2lLbUkrVDhaZGsKKzhmMjFEdWx3b3BmZEI5R0hlSm5wN0svcnVKUkJ5MVdvM1lXbko5dUlYZFhZOUFqS0E9PQotLS0tLUVORCBDRVJUSUZJQ0FURS0tLS0tCg==
        client-key-data: LS0tLS1CRUdJTiBSU0EgUFJJVkFURSBLRVktLS0tLQpNSUlFcEFJQkFBS0NBUUVBeVU2NTd0TVRpVHFCeFk5ZVpqT0ZNWURKNXFSVE16YUxTemJ1bkttOUUyczZTV3daCjI2STVsTTVXMW1BcktLUDljc0RaUE1ZTXBDYitpVGZ6M09FOGYzc2c0M21oSGgwUXFkRm8xdFZPcmZYMkZNZE8KQ3AyRkFhellMRXVVZ1FSU0MrSllXYk9keml3aHFNUHlJNCsxVE5ldlYrUFRwNGViWUdUakxKTmMwWTVIWkxINAp4bTU1U2VIc2hJQU45U2tUV042QVFoeFpVWUNLWlJCcnU5M0ZjTVdmc3FNSWM1MGIwcUNCZC9HelNpVEtDRFhSCm1Qd1ExWE1xblNLem1MdjkwTkRPWGI2dUh4YkttYlhsVHNQelQ2Q055VERQTnVkK29YWGRJbEdoWmllQTIvc24KcitYTDRuTzZSUDR4cFQyeUlKQURtdHRiRDNnV3NIdU1WUWFvOHdJREFRQUJBb0lCQVFDbUo4bkYrd2liOHVPYgorZnJ6cGtDZ25HbUpha2FWOWNaQkhhVVRQL0trN1pOZGVORmEvR3BFalk4VlFLayswU1JuckE5aVh5R2Q5K1dOCnd0WVFrUVFMUU1qam1NZklnRHI1djdPbDVzZ2JRL0dLTXZzU1BmUERiek82VStQT0hZL082Vkw5THdqb1hIcW4KdnB2RWlHQWZmY0xuYTArT2JwcHJsTG9CVjl4N3hWajBTcy9PSkJvRzVQbzJXTk0ydDB3a3hHUUQzMzF3bitzQQo0eGNkblU1UVBtdEkybVlrY01Cbnc4MDUra2xpUld0c3lrSWVsR2tWS00vM1NiVzJDeU1HUE5ReGpyaWd3NDVMCk44YThnemo1VDhFWnV1T2dYWHY4Z1F3azhhOHNjd25nOVZXVW0xbUJaN1k1cEp0NEJVZEY2Q2NuSTZ0RS9sa2MKVng4RkpYVUpBb0dCQU5IUC9PQWVTWC90cWIyeG1sRm5rNHZmbGxieFpLVU1PSnFmT2RoZ081bWdjV0NSRkZBRwpja1pkMFk1cjZqc3dMNVhtaWRLakpsVmJsaWY3dDFsY3plZUFXUHlOZUhvelRsSzU4UmF1L2MyTVdmeFBFSXU2ClNKR3UxdTlmTzZXZVUzbjVqcUluQWpiMzlyTEM3SDkyZVhzTDlpQy9xT0d3eXVhQ1M5clJUc010QW9HQkFQV2YKY1NqV2Q2dXBIQjQrZUJESkx4VmU1bDkyMzhPYWlxbkU3ZGlERHdhYWd1Y1lHbkRSZ09NaGlzOUNtNUxuZGticQpDRVNHZ01BMHQwZ09vb1puajN5eWlaZlkwNUI4K21NcEhtMnJ3SnpoS3dEemI4S09QcFB0Z212YVBROVpaWitjCkltejJvTzFvTjlURjVid0dBK3lFZy9EYnpJNUxVVHFQZWxDd2J6Q2ZBb0dCQU1FT1gwR2R2TVhBMnRJWUxNWEEKeDR3SjFOejFTMFZ2SkZwcUxxREJrN1c5WXZXWEtSaWxoZHJua3Q0NHdCTnNPQ3ozTDFRcEdTbXJsMVA5RXUxZwpMbnBZcUFqaTU3dVJuLzBRNlJ5Vk1pWkRnYjFleHZ1N0VmRXk3c1RkWFJYOHhCVFZJNEJpNG0vUDVDa0NvUGg3Ci9EWFRnTXNMY0FzVFVPK2Zic3JPazJtVkFvR0FXQitKUU9hWmJ0d3dlMlZjUEdHQjQvLzFWVURZRFZ5djdUTDcKUnBmVzF6NnVRbTB5WjFHekZVcGVlL2ZneXpjQ0IzVkYzQmdKcjJ2NmFmN2VMcXlQSFdVTTJvN3ZjTUoyTHdkOApwRXBmdzZsQmZZalppd3J2eHJFSy90a0EyVFh3c1BBYXBjOWljMnJWeFIvdlNhTTYyeXU4RHJrOVRid1YrNVdvCmc3U1pYKzhDZ1lBcG9PUDZzR0MyYjVNR1BWWjdWUW1nS2JtRnU1WnAwQnFMV1U4dno2VnltNnZGMWV2NmpKQy8KY0V3UEpYM2xpWkJRbE9mUm54QmNIRW9aZkZyQUZDcWZCbEcyR0lCVkpiQ0dzWnZKQ1AvVUgwWFpWazk3VHFjcgpTdXhpaFlsR3FVb0lyWDVPSEFGeHUwdHRjYmsrbm8xZUFLUHYvZmx0QlovNmRMc2tFcC94aVE9PQotLS0tLUVORCBSU0EgUFJJVkFURSBLRVktLS0tLQo=
--- #! appsettings.json ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: gzctf-config
data:
  appsettings.json: |-
    {
    "AllowedHosts": "*",
    "ConnectionStrings": {
        "Database": "Host=gzctf-db:5432;Database=gzctf;Username=postgres;Password=postgress",
        "RedisCache": "gzctf-redis:6379,password=rediss"
    },
    "Logging": {
      "LogLevel": {
      "Default": "Information",
      "Microsoft": "Warning",
      "Microsoft.Hosting.Lifetime": "Information"
      }
    },
    "EmailConfig": {
      "SendMailAddress": "admin@birkenwald-lab.top",
      "UserName": "admin@birkenwald-lab.top",
      "Password": "BirkenWaldF802",
      "Smtp": {
        "Host": "smtp.qcloudmail.com",
        "Port": 465
      }
    },
    "XorKey": "birkenwald-gzctf-2023",
    "ContainerProvider": {
        "Type": "Docker",
        "PortMappingType": "Default",
        "EnableTrafficCapture": false,
        "PublicEntry": "192.168.227.83",
        "DockerConfig": {
        "SwarmMode": false,
        "Uri": "unix:///var/run/docker.sock"
        }
    },
    "RequestLogging": true,
    "DisableRateLimit": false
    }
--- #! gzctf-files-pv PV
apiVersion: v1
kind: PersistentVolume
metadata:
  name: gzctf-files-pv
spec:
  capacity:
    storage: 20Gi
  accessModes:
    - ReadWriteOnce
  nfs:
    server: 192.168.227.82
    path: /gzctf/files
--- #! gzctf-dv-pv PV
apiVersion: v1
kind: PersistentVolume
metadata:
  name: gzctf-db-pv
spec:
  capacity:
    storage: 20Gi
  accessModes:
    - ReadWriteOnce
  nfs:
    server: 192.168.227.82
    path: /gzctf/db
--- #! gzctf-files PVC
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: gzctf-files
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
  volumeName: gzctf-files-pv
--- #! gzctf-db PVC
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: gzctf-db
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
  volumeName: gzctf-db-pv

--- #! gzctf-db 实例部署
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gzctf-db
  labels:
    app: gzctf-db
spec:
  replicas: 1
  selector:
    matchLabels:
      app: gzctf-db
  template:
    metadata:
      labels:
        app: gzctf-db
    spec:
      nodeSelector:
        kubernetes.io/hostname: k8s-worker
      containers:
        - name: gzctf-db
          image: postgres:alpine
          ports:
            - containerPort: 5432
              name: postpres
          env:
            - name: POSTGRES_PASSWORD
              value: postgress
          volumeMounts:
            - name: gzctf-db
              mountPath: /var/lib/postgresql/data
          resources:
            requests:
              cpu: 500m
              memory: 512Mi
      volumes:
        - name: gzctf-db
          persistentVolumeClaim:
            claimName: gzctf-db
--- #! gzctf-redis 实例部署
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gzctf-redis
  labels:
    app: gzctf-redis
spec:
  replicas: 1
  selector:
    matchLabels:
      app: gzctf-redis
  template:
    metadata:
      labels:
        app: gzctf-redis
    spec:
      nodeSelector:
        kubernetes.io/hostname: k8s-worker1
      containers:
        - name: gzctf-redis
          image: redis:alpine
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 6379
              name: redis
          resources:
            requests:
              cpu: 100m
              memory: 256Mi

--- #! gzctf 实例部署
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gzctf
  labels:
    app: gzctf
spec:
  replicas: 3
  selector:
    matchLabels:
      app: gzctf
  template:
    metadata:
      labels:
        app: gzctf
    spec:

      containers:
        - name: gzctf
          image: gztime/gzctf:latest
          imagePullPolicy: IfNotPresent
          env:
            - name: GZCTF_ADMIN_PASSWORD
              value: birkenwald-admin
          ports:
            - containerPort: 80
              name: http
          volumeMounts:
            - name: gzctf-files
              mountPath: /app/files
            - name: gzctf-config
              mountPath: /app/appsettings.json
              subPath: appsettings.json
            - name: gzctf-k8sconfig
              mountPath: /app/k8sconfig.yaml
              subPath: k8sconfig
            - name: dockersock
              mountPath: /var/run/docker.sock
          resources:
            requests:
              cpu: 1000m
              memory: 384Mi
      volumes:
        - name: gzctf-files
          persistentVolumeClaim:
            claimName: gzctf-files
        - name: gzctf-config
          configMap:
            name: gzctf-config
        - name: gzctf-k8sconfig
          secret:
            secretName: gzctf-k8sconfig
        - name: dockersock
          hostPath:
            path: /var/run/docker.sock

--- #! gzctf Service
apiVersion: v1
kind: Service
metadata:
  name: gzctf
  annotations:
    traefik.ingress.kubernetes.io/service.sticky.cookie: "true"
    traefik.ingress.kubernetes.io/service.sticky.cookie.name: "LB_Session"
    traefik.ingress.kubernetes.io/service.sticky.cookie.httponly: "true"
spec:
  type: NodePort
  selector:
    app: gzctf
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
      nodePort: 30080

--- #! gzctf-db Service
apiVersion: v1
kind: Service
metadata:
  name: gzctf-db
spec:
  selector:
    app: gzctf-db
  ports:
    - protocol: TCP
      port: 5432
      targetPort: 5432

--- #! gzctf-redis Service
apiVersion: v1
kind: Service
metadata:
  name: gzctf-redis
spec:
  selector:
    app: gzctf-redis
  ports:
    - protocol: TCP
      port: 6379
      targetPort: 6379
```

### `traefik ingress controller安装`

`1. 安装 helm`
``` bash
snap install helm
# Helm 是 Kubernetes 的包管理器。包管理器类似于我们在 Ubuntu 中使用的apt、Centos中使用的yum 或者Python中的 pip 一样，能快速查找、下载和安装软件包。Helm 由客户端组件 helm 和服务端组件 Tiller 组成, 能够将一组K8S资源打包统一管理, 是查找、共享和使用为Kubernetes构建的软件的最佳方式。
```
`2. 添加traefik库`
``` bash
helm repo add traefik https://helm.traefik.io/traefik
```
`3. 更新库`
``` bash
helm repo update 
```
`4. 部署traefik`
``` bash
helm install traefik traefik/traefik -n traefik \
--set service.annotations."traefik\.ingress\.kubernetes\.io/rule-type"=PathPrefixStrip \
--set service.annotations."traefik\.ingress\.kubernetes\.io/rule-value"="/(dashboard|api)" \
--set entryPoints.web.address=:80 \
--set entryPoints.web.http.redirections.entryPoint.to=websecure \
--set entryPoints.web.http.redirections.entryPoint.scheme=https
```
`5. 部署 gzctf-tls`
``` bash
kubectl create secret tls my-tls-secret --key /path/to/private/xxx.key --cert /path/to/xxx.crt # 至于ssl加密的证书需要自己获取，我们www.birkenwald-lab.top的证书在ctfd服务器里，进去就能找到
```

`6. 部署ingressroute，路由访问请求`
``` yaml
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: my-ingressroute
spec:
  entryPoints:
    - web
  routes:
    - match: Host("example.com")
      kind: Rule
      services:
        - name: gzctf
          port: 80
          kind: Service
  tls:
    secretName: gzctf-tls
```

**PS: 本文由l1Akr创作，仅供内部交流，请勿转发到其他网站！！！！！！**

