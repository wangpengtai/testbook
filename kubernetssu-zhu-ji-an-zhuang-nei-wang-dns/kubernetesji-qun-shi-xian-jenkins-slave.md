## 一、安装Jenkins



我们将使用[Jenkins的`kubernetes`插件](https://github.com/jenkinsci/kubernetes-plugin)通过配置动态代理来适应当前的工作负载，从而在集群上扩展Jenkins。

该插件将通过基于特定Docker镜像启动代理来为每个构建创建Kubernetes Pod。

构建完成后，Jenkins将删除Pod以节省资源。

代理将使用JNLP（Java网络启动协议）启动，因此我们的容器将能够在启动并运行后自动连接到Jenkins主服务器。![](https://rancher.com/img/blog/2018/scaling-jenkins/01-rancher-jenkins-master-slave-architecture.png)

### 1. 存储服务器

找一台服务器搭建一台nfs服务器&lt;&lt;详见Ubuntu16.04 安装nfs&gt;&gt;

系统：Ubuntu 16.04

IP：172.18.1.13

```
apt install nfs-common nfs-kernel-server -y

#配置挂载信息
cat /etc/exports

/data/k8s *(rw,sync,no_root_squash)

#给目录添加权限

chmod -R 777 /data/k8s

#启动

/etc/init.d/nfs-kernel-server start

#开机启动

systemctl enable nfs-kernel-server
```

### 2. kubernetes集群安装Jenkins

```shell
#该目录下是jenkins.mytest.io为测试域名的自签密钥
ls jenkins.mytest.io/
cacerts.pem  cacerts.srl  cakey.pem  create_self-signed-cert.sh  jenkins.mytest.io.crt  jenkins.mytest.io.csr  jenkins.mytest.io.key  openssl.cnf  tls.crt  tls.key
---
cd jenkins.mytest.io 
#创建Jenkins所在的namespace
kubectl create namespace kube-ops

#将密文添加到kube-ops里面
# 服务证书和私钥密文
kubectl -n kube-ops create \
    secret tls tls-jenkins-ingress \
    --cert=./tls.crt \
    --key=./tls.key
# ca证书密文
kubectl -n kube-ops create secret \
    generic tls-ca \
    --from-file=cacerts.pem
```

### 3. 创建Jenkins

**cat jenkins-pvc.yaml**

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: jenkins-pv
spec:
  capacity:
    storage: 20Gi
  accessModes:
  - ReadWriteMany
  persistentVolumeReclaimPolicy: Delete
  nfs:
    server: 172.18.1.13
    path: /data/k8s

---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: jenkins-pvc
  namespace: kube-ops
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 20Gi
```

**cat rbac.yaml**

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: jenkins
  namespace: kube-ops

---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: jenkins
rules:
  - apiGroups: ["extensions", "apps"]
    resources: ["deployments"]
    verbs: ["create", "delete", "get", "list", "watch", "patch", "update"]
  - apiGroups: [""]
    resources: ["services"]
    verbs: ["create", "delete", "get", "list", "watch", "patch", "update"]
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["create","delete","get","list","patch","update","watch"]
  - apiGroups: [""]
    resources: ["pods/exec"]
    verbs: ["create","delete","get","list","patch","update","watch"]
  - apiGroups: [""]
    resources: ["pods/log"]
    verbs: ["get","list","watch"]
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get"]

---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: jenkins
  namespace: kube-ops
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: jenkins
subjects:
  - kind: ServiceAccount
    name: jenkins
    namespace: kube-ops
```

**cat jenkins.yaml**

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: jenkins
  namespace: kube-ops
spec:
  template:
    metadata:
      labels:
        app: jenkins
    spec:
      terminationGracePeriodSeconds: 10
      serviceAccount: jenkins
      containers:
      - name: jenkins
        image: jenkins/jenkins:lts
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 8080
          name: web
          protocol: TCP
        - containerPort: 50000
          name: agent
          protocol: TCP
        resources:
          limits:
            cpu: 1000m
            memory: 1Gi
          requests:
            cpu: 500m
            memory: 512Mi
        livenessProbe:
          httpGet:
            path: /login
            port: 8080
          initialDelaySeconds: 60
          timeoutSeconds: 5
          failureThreshold: 12
        readinessProbe:
          httpGet:
            path: /login
            port: 8080
          initialDelaySeconds: 60
          timeoutSeconds: 5
          failureThreshold: 12
        volumeMounts:
        - name: jenkinshome
          subPath: jenkins
          mountPath: /var/jenkins_home
        env:
        - name: LIMITS_MEMORY
          valueFrom:
            resourceFieldRef:
              resource: limits.memory
              divisor: 1Mi
        - name: JAVA_OPTS
          value: -Xmx$(LIMITS_MEMORY)m -XshowSettings:vm -Dhudson.slaves.NodeProvisioner.initialDelay=0 -Dhudson.slaves.NodeProvisioner.MARGIN=50 -Dhudson.slaves.NodeProvisioner.MARGIN0=0.85 -Duser.timezone=Asia/Shanghai
      securityContext:
        fsGroup: 1000
      volumes:
      - name: jenkinshome
        persistentVolumeClaim:
          claimName: jenkins-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: jenkins
  namespace: kube-ops
  labels:
    app: jenkins
spec:
  selector:
    app: jenkins
  type: NodePort
  ports:
  - name: web
    port: 8080
    targetPort: web
    nodePort: 30002
  - name: agent
    port: 50000
    targetPort: agent

---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: jenkins-lb
  namespace: kube-ops
spec:
  tls:
    - secretName: tls-jenkins-ingress
  rules:
  - host: jenkins.mytest.io
    http:
      paths:
      -  backend:
          serviceName: jenkins
          servicePort: 8080
```

创建Jenkins

```shell
kubectl create -f pvc.yaml
kubectl create -f rbac.yaml
kubectl create -f jenkins.yaml
```

---

## 二、jenkins配置

在/etc/hosts配置域名解析

```
kube-ip jenkins.mytest.io
```

### 1. 初始化配置

打开[https://jenkins.mytes.io](https://jenkins.mytes.io)![](https://github.com/wangpengtai/testbook/blob/master/images/kubernetes install jenkins slave/images/jenkins01.png?raw=true)

安装插件，选择默认即可

![](https://github.com/wangpengtai/testbook/blob/master/images/kubernetes install jenkins slave/images/jenkins02.png?raw=true)

### 2. 插件配置

采用Jenkins里面的kubernetes插件，让Jenkins可以调用kubernetes生成Jenkins-slave

![](https://github.com/wangpengtai/testbook/blob/master/images/kubernetes install jenkins slave/images/jenkins03.png?raw=true)

#### 2.1  安装kubernetes插件

Manage Jenkins -&gt; Manage Plugins -&gt; Available -&gt; Kubernetes plugin 勾选安装即可。![](https://github.com/wangpengtai/testbook/blob/master/images/kubernetes install jenkins slave/images/jenkins04.png?raw=true)

---

#### 2.2 配置kubernetes插件功能

* Manage Jenkins —&gt; Configure System —&gt; \(拖到最下方\)Add a new cloud —&gt; 选择 Kubernetes，然后填写 Kubernetes 和 Jenkins 配置信息。

* kubernetes地址采用了kube的服务器发现[https://kubernetes.default.svc.cluster.local](https://kubernetes.default.svc.cluster.local)

* namespace填kube-ops，然后点击Test Connection，如果出现 Connection test successful 的提示信息证明 Jenkins 已经可以和 Kubernetes 系统正常通信

* Jenkins URL 地址：[http://jenkins.kube-ops.svc.cluster.local:8080](http://jenkins.kube-ops.svc.cluster.local:8080)

![](https://github.com/wangpengtai/testbook/blob/master/images/kubernetes install jenkins slave/images/jenkins05.png?raw=true)![](https://github.com/wangpengtai/testbook/blob/master/images/kubernetes install jenkins slave/images/jenkins06.png?raw=true)

另外需要注意，如果这里 Test Connection 失败的话，很有可能是权限问题，这里就需要把我们创建的 jenkins 的 serviceAccount 对应的 secret 添加到这里的 Credentials 里面。

---

#### 2.3 配置 kubernetes Pod Template

其实就是配置 Jenkins Slave 运行的 Pod 模板，命名空间我们同样是用 kube-ops，Labels 这里也非常重要，对于后面执行 Job 的时候需要用到该值，然后我们这里使用的是 cnych/jenkins:jnlp 这个镜像，这个镜像是在官方的 jnlp 镜像基础上定制的，加入了 kubectl 等一些实用的工具。![](https://github.com/wangpengtai/testbook/blob/master/images/kubernetes install jenkins slave/images/jenkins07.png?raw=true)

---

#### 2.4 添加容器的挂载卷

另外需要注意我们这里需要在下面挂载两个主机目录，一个是 /var/run/docker.sock，该文件是用于 Pod 中的容器能够共享宿主机的 Docker，这就是大家说的 docker in docker 的方式，Docker 二进制文件我们已经打包到上面的镜像中了，另外一个目录下 /root/.kube 目录，我们将这个目录挂载到容器的 /home/jenkins/.kube 目录下面这是为了让我们能够在 Pod 的容器中能够使用 kubectl 工具来访问我们的 Kubernetes 集群，方便我们后面在 Slave Pod 部署 Kubernetes 应用。

![](https://github.com/wangpengtai/testbook/blob/master/images/kubernetes install jenkins slave/images/jenkins08.png?raw=true)

---

#### 2.5 添加账号

另外一些同学在配置了后运行 Slave Pod 的时候出现了权限问题，因为 Jenkins Slave Pod 中没有配置权限，所以需要配置上 ServiceAccount，在 Slave Pod 配置的地方点击下面的高级，添加上对应的 ServiceAccount 即可：

测试的时候不添加账号会告知没有权限

![](https://github.com/wangpengtai/testbook/blob/master/images/kubernetes install jenkins slave/images/jenkins10.png?raw=true)

在容器模板高级里面添加kubernetes集群中创建的jenkins账号

![](https://github.com/wangpengtai/testbook/blob/master/images/kubernetes install jenkins slave/images/jenkins09.png?raw=true)

---

## 三、测试

创建一个测试任务

![](https://github.com/wangpengtai/testbook/blob/master/images/kubernetes install jenkins slave/images/jenkins11.png?raw=true)

在pipeline的框里面添加一下内容![](https://github.com/wangpengtai/testbook/blob/master/images/kubernetes install jenkins slave/images/jenkins12.png?raw=true)

```js
def label = "jnlp-slave"
podTemplate(inheritFrom: 'jnlp-slave', instanceCap: 0, label: 'jnlp-slave', name: '', namespace: 'kube-ops', nodeSelector: '', podRetention: always(), serviceAccount: '', workspaceVolume: emptyDirWorkspaceVolume(false), yaml: '') {
    node(label) {
        container('jnlp-slave'){
            stage('Run shell') {
                sh 'docker info'
                sh 'kubectl get pods -n kube-ops'
            }
        }
    }
}
```

开始构建任务![](https://github.com/wangpengtai/testbook/blob/master/images/kubernetes install jenkins slave/images/jenkins13.png?raw=true)

构建任务输出

```
Started by user admin
Running in Durability level: MAX_SURVIVABILITY
[Pipeline] Start of Pipeline
[Pipeline] podTemplate
[Pipeline] {
[Pipeline] node
Still waiting to schedule task
‘Jenkins’ doesn’t have label ‘jnlp-slave’
Agent jnlp-slave-tbdnl is provisioned from template Kubernetes Pod Template
Agent specification [Kubernetes Pod Template] (jnlp-slave):
* [jnlp-slave] cnych/jenkins:jnlp
Running on jnlp-slave-tbdnl in /home/jenkins/workspace/test-jnlp-slave
[Pipeline] {
[Pipeline] container
[Pipeline] {
[Pipeline] stage
[Pipeline] { (Run shell)
[Pipeline] sh
docker info
Containers: 15
Running: 12
Paused: 0
Stopped: 3
Images: 12
Server Version: 18.09.6
Storage Driver: overlay2
Backing Filesystem: extfs
Supports d_type: true
Native Overlay Diff: false
Logging Driver: json-file
Cgroup Driver: cgroupfs
Plugins:
Volume: local
Network: bridge host macvlan null overlay
Log: awslogs fluentd gcplogs gelf journald json-file local logentries splunk syslog
Swarm: inactive
Runtimes: runc
Default Runtime: runc
Init Binary: docker-init
containerd version: bb71b10fd8f58240ca47fbb579b9d1028eea7c84
runc version: 2b18fe1d885ee5083ef9f0838fee39b62d653e30
init version: fec3683
Security Options:
apparmor
seccomp
Profile: default
Kernel Version: 4.15.0-1049-azure
Operating System: Ubuntu 16.04.6 LTS
OSType: linux
Architecture: x86_64
CPUs: 2
Total Memory: 7.768GiB
Name: test-kube-node-04
ID: YFTJ:FVHK:TAF3:HTAJ:HJ2A:5SFW:73RW:VQY5:Y64U:UGIR:KMJ2:XPRL
Docker Root Dir: /var/lib/docker
Debug Mode (client): false
Debug Mode (server): false
Registry: 
https://index.docker.io/v1/

Labels:
Experimental: false
Insecure Registries:
127.0.0.0/8
Registry Mirrors:

https://kv3qfp85.mirror.aliyuncs.com/

Live Restore Enabled: false
WARNING: No swap limit support
[Pipeline] sh
kubectl get pods -n kube-ops
NAME                      READY     STATUS    RESTARTS   AGE
jenkins-6b874b8d7-q28h4   1/1       Running   0          3h
jnlp-slave-tbdnl          2/2       Running   0          15s
[Pipeline] }
[Pipeline] // stage
[Pipeline] }
[Pipeline] // container
[Pipeline] }
[Pipeline] // node
[Pipeline] }
[Pipeline] // podTemplate
[Pipeline] End of Pipeline
Finished: SUCCESS
```

---

## 结束语

根据上面步骤（大部分都是根据阳明博客上的《基于 Jenkins 的 CI/CD\(一\)》思路跟进），但是由于环境和自己认知问题，会出现各种出错，憋了一天，没什么进展。

后来只能根据`kubectl -n kube-ops logs -f jenkins-xxxxx`的命令一点点查出来的，搜过很多帖子大概思路一致，但是无法解决本质问题，如果跑不起来，再高端也是个没有用，后来根据kubernetes-plugin的github，具体读了遍结合自己报的错一点一点调整过来，总算搞出来了。

---

**具体参考了以下几遍优秀的文章:**

[基于 Jenkins 的 CI/CD\(一\)（阳明老师的文章很棒）](https://www.qikqiak.com/post/kubernetes-jenkins1)

[kubernetes Jenkins gitlab搭建CI/CD环境 \(二\)](https://www.sudops.com/kubernetes-jenkins-gitlab-ci-cd-env-2.html)

[GitHub-Jenkins-kubernetes-plugin：jenkinsci/kubernetes-plugin](https://github.com/jenkinsci/kubernetes-plugin)

[rancher官网: 在Kubernetes上部署和扩展Jenkins](https://rancher.com/blog/2018/2018-11-27-scaling-jenkins/)

