# 基于kubernetes集群的devops实践

> ## **GitHub:**[**Helm Chart for Harbor**](https://github.com/goharbor/harbor-helm)

## 1. 添加harbor的helm库

```text
helm repo add harbor https://helm.goharbor.io
```

## 2. 将harbor下载到本地

```text
helm fetch harbor/harbor
tar xf harbor-1.1.1.tgz
```

## 3. 自定义配置

默认的values.yaml基本不需要多大改动，只有极个别的需要自定义修改

具体改什么需要根据自动需求，可以查看GitHub上面的配置列表[configuration](https://github.com/goharbor/harbor-helm/blob/master/README.md#configuration)，根据自己的特殊需求创建一个新的values文件，覆盖掉之前的值

```yaml
cat new-values.yaml
expose:
  type: ingress
  tls:
    enabled: true
  ingress:
    hosts:
      core: harbor.mytest.io
      notary: notary.mytest.io

externalURL: https://harbor.mytest.io

persistence:
  enabled: true
  resourcePolicy: "keep"
  persistentVolumeClaim:
    registry:
      storageClass: "harbor-data"
    chartmuseum:
      storageClass: "harbor-data"
    jobservice:
      storageClass: "harbor-data"
    database:
      storageClass: "harbor-data"
    redis:
      storageClass: "harbor-data"
```

## 4. 创建一个harbor的存储类

存储类见[《kubernetes集群使用nfs-client实现storageclass》](https://blog.51cto.com/wangpengtai/2418609)

```yaml
cat harbor-data-sc.yaml

apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: harbor-data
provisioner: fuseim.pri/ifs
```

## 5. 创建harbor

```text
kubectl create -f harbor-data-sc.yaml
helm install --name harbor -f new-values.yaml --namespace kube-ops ./harbor
```

## 6. 查看ingress

```text
kubectl get ingresses. -n kube-ops
NAME                    HOSTS                               ADDRESS                                                                                                 PORTS     AGE
harbor-harbor-ingress   harbor.mytest.io,notary.mytest.io   172.18.1.14,.....,172.18.1.9   80, 443   101m
```

## 7. 本地访问

配置本地/etc/hosts文件

```text
kube-ip harbor.mytest.io
```

使用浏览器访问`https://harbor.mytest.io`

账号：`admin`

密码：`Harbor12345`

