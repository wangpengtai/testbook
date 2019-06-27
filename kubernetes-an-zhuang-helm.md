> **参考**
>
> [https://www.cnrancher.com/docs/rancher/v2.x/cn/installation/ha-install/](https://www.cnrancher.com/docs/rancher/v2.x/cn/installation/ha-install/)
>
> 注意
>
> helm使用需要kubectl，点击了解安装和配置[kubectl](https://www.cnrancher.com/docs/rancher/v2.x/cn/install-prepare/kubectl/ "kubectl")

Helm是Kubernetes首选的包管理工具。Helm`charts`为Kubernetes YAML清单文档提供模板语法。使用Helm，我们可以创建可配置的部署，而不仅仅是使用静态文件。有关创建自己的`charts`的更多信息，请查看[https://helm.sh/](https://helm.sh/)文档。Helm有两个部分：Helm客户端\(helm\)和Helm服务端\(Tiller\)。

### 1. 配置Helm客户端访问权限

Helm在集群上安装`tiller`服务以管理`charts`. 由于RKE默认启用RBAC, 因此我们需要使用`kubectl`来创建一个`serviceaccount`，`clusterrolebinding`才能让`tiller`具有部署到集群的权限。

* 在`kube-system`命名空间中创建`ServiceAccount`

* 创建`ClusterRoleBinding`以授予`tiller`帐户对集群的访问权限

* `helm`初始化`tiller`服务

```shell
kubectl -n kube-system create serviceaccount tiller

kubectl create clusterrolebinding tiller \
--clusterrole cluster-admin --serviceaccount=kube-system:tille
```

### 2. 安装Helm客户端

这里采用二进制安装，可以在`helm`官网上下载[Helm](https://github.com/helm/helm/releases),如果速度不理想可以使用rancher自带加速下载[Helm](https://www.cnrancher.com/docs/rancher/v2.x/cn/install-prepare/download/helm/)。

```shell
wget https://www.cnrancher.com/download/helm/helm-v2.14.1-linux-amd64.tar.gz
tar xf helm-v2.14.1-linux-amd64.tar.gz
```

`helm`在解压后的目录中找到二进制文件，并将其移动到所需的位置。

```shell
sudo mv linux-amd64/helm /usr/local/bin/helm && chmod +x /usr/local/bin/helm
```

### 3. 安装Helm Server\(Tiller\)

Helm的服务器端部分Tiller,通常运行在Kubernetes集群内部。但是对于开发，它也可以在本地运行，并配置为与远程Kubernetes集群通信。

安装`tiller`到集群中最简单的方法就是运行`helm init`。这将验证`helm`本地环境设置是否正确\(并在必要时进行设置\)。然后它会连接到`kubectl`默认连接的K8S集群\(`kubectl config view`\)。一旦连接，它将安装`tiller`到`kube-system`命名空间中。

`helm init`自定义参数:

* `--canary-image` 参数安装金丝雀版本;

* `--tiller-image`安装特定的镜像\(版本\);

* `--kube-context` 使用安装到特定集群;

* `--tiller-namespace` 用一个特定的命名空间`(namespace)`安装;

```
注意:
1、RKE默认启用RBAC,所以在安装tiller时需要指定ServiceAccount。
2、helm init在缺省配置下，会去谷歌镜像仓库拉取gcr.io/kubernetes-helm/tiller镜像，在Kubernetes集群上安装配置Tiller；由于在国内可能无法访问gcr.io、storage.googleapis.com等域名，可以通过--tiller-image指定私有镜像仓库镜像。 
3、helm init在缺省配置下，会利用https://kubernetes-charts.storage.googleapis.com作为缺省的stable repository地址,并去更新相关索引文件。在国内可能无法访问storage.googleapis.com地址, 可以通过--stable-repo-url指定chart国内加速镜像地址。 
4、如果您是离线安装Tiller, 假如没有内部的chart仓库, 可通过添加--skip-refresh参数禁止Tiller更新索引。
```

在rancher中安装Tiller

```shell
wangpeng@test-kube-master-01:~$ helm init --service-account tiller --skip-refresh \
> --tiller-image registry.cn-shanghai.aliyuncs.com/rancher/tiller:v2.14.1
Creating /home/wangpeng/.helm 
Creating /home/wangpeng/.helm/repository 
Creating /home/wangpeng/.helm/repository/cache 
Creating /home/wangpeng/.helm/repository/local 
Creating /home/wangpeng/.helm/plugins 
Creating /home/wangpeng/.helm/starters 
Creating /home/wangpeng/.helm/cache/archive 
Creating /home/wangpeng/.helm/repository/repositories.yaml 
Adding stable repo with URL: https://kubernetes-charts.storage.googleapis.com 
Adding local repo with URL: http://127.0.0.1:8879/charts 
$HELM_HOME has been configured at /home/wangpeng/.helm.

Tiller (the Helm server-side component) has been installed into your Kubernetes Cluster.

Please note: by default, Tiller is deployed with an insecure 'allow unauthenticated users' policy.
To prevent this, run `helm init` with the --tiller-tls-verify flag.
For more information on securing your installation see: https://docs.helm.sh/using_helm/#securing-your-helm-installation
```

查看`helm`情况

```shell
wangpeng@test-kube-master-01:~$ helm version
Client: &version.Version{SemVer:"v2.14.1", GitCommit:"5270352a09c7e8b6e8c9593002a73535276507c0", GitTreeState:"clean"}
Server: &version.Version{SemVer:"v2.14.1", GitCommit:"5270352a09c7e8b6e8c9593002a73535276507c0", GitTreeState:"clean"}
```

### 4. 添加Chart仓库地址

使用`helm repo add`命令添加`Rancher chart`仓库地址,访问`Rancher tag`和`Chart`版本。

替换`<CHART_REPO>`为您要使用的`Helm`仓库分支\(即`latest`或`stable`）。

```shell
helm repo list
NAME    URL                                             
stable  https://kubernetes-charts.storage.googleapis.com
local   http://127.0.0.1:8879/charts
```

添加`rancher`的`charts`库：

```shell
helm repo add rancher-stable \
https://releases.rancher.com/server-charts/stable
```

添加azure的charts库:

```shell
helm repo add azure-stable \
http://mirror.azure.cn/kubernetes/charts/
```

查看`helm`的`repo`源

```shell
helm repo list
NAME            URL                                              
stable          https://kubernetes-charts.storage.googleapis.com 
local           http://127.0.0.1:8879/charts                     
rancher-stable  https://releases.rancher.com/server-charts/stable
```



