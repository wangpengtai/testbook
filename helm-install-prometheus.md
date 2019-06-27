### 1. 给prometheus创建持久卷

prometheus的service和altermanger需要使用到持久性存储，可以不用为了安全，在没有nfs条件的情况下，使用了映射本地目录的方案。

```
mkdir -pv /data/{data01,data02}
```

在`kubernetes`集群里面创建`pv1`、`pv2`

```
cat >> pv1.yaml<< EOF
apiVersion: v1
kind: PersistentVolume
metadata:
  name: data01
spec:
  capacity:
    storage: 2Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Recycle
  local:
    path: /data/data01 #创建需要挂载的本地目录
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - base-test-kube-master-01 #修改为需要挂载的node节点名称
EOF
---
cat >> pv2.yaml <<EOF
apiVersion: v1
kind: PersistentVolume
metadata:
  name: data02
spec:
  capacity:
    storage: 8Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Recycle
  local:
    path: /data/data02
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - base-test-kube-master-01
EOF
---

kubectl create -f pv1.yaml

kubectl create -f pv2.yaml
```

### 2. helm安装prometheus

将stable上面prometheus的charts安装包下载到本地。

```
helm fetch stable/prometheus
```

修改prometheus下面的values.yaml里面的值

```
alertmanager:
  ingress:
    enabled: true
    hosts: [alertmanager.domain.com]
  persistentVolume:
    enabled: true
---  
server:
  ingress:
    enabled: true
    hosts: [prometheus.domain.com]
  persistentVolume:
    enabled: true
---
pushgateway:
  ingress:
    enabled: true
    hosts: [pushgateway.domain.com]
修改完毕values.yaml保存后，使用helm安装prometheus
```

```
 helm install --namespace minitor --name prometheus ./prometheus
```

### 3. helm安装grafana

修改grafana以下的values.yaml值

```
ingress:
  enabled: true
  annotations: {kubernetes.io/ingress.class: traefik,traefik.frontend.rule.type: PathPrefixStrip}
  path: /
  hosts:
    - grafana.domain.com
adminUser: admin
adminPassword: "admin123qwe"
```

helm安装grafana

```
helm install --namespace monitor --name grafana ./grafana
```

### 4.  配置grafana

![](https://note.youdao.com/yws/api/personal/file/WEBcd8f97199ac34653264a0a49d118d477?method=download&shareKey=0bccc95afb4915c076838e53b774c9b2)

### 5. 导入grafana模板

选取“+”图标，import进去kubernetes-tool/monitor/dashboard下的面板json文件，或者直接import一个 [https://grafana.com ](https://grafana.com上的模版，即可看到监控面板的情况（有的有一些问题，需要自行试验、选择）。也可以自己设计面板、然后保存起来，或者分享给别人使用。)上的模版，即可看到监控面板的情况（有的有一些问题，需要自行试验、选择）。也可以自己设计面板、然后保存起来，或者分享给别人使用。

