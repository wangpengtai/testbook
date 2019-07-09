### **GitHub:**[**Helm Chart for Harbor**](https://github.com/goharbor/harbor-helm)

### 1. 添加harbor的helm库

```shell
helm repo add harbor https://helm.goharbor.io
```

### 2. 将harbor下载到本地

```shell
helm fetch harbor/harbor
tar xf harbor-1.1.1.tgz
```

### 3. 自定义配置

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

### 4. 创建一个harbor的存储类

存储类见[《kubernetes集群使用nfs-client实现storageclass》](https://blog.51cto.com/wangpengtai/2418609)

```yaml
cat harbor-data-sc.yaml

apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: harbor-data
provisioner: fuseim.pri/ifs
```

### 5. 创建harbor

```shell
kubectl create -f harbor-data-sc.yaml
helm install --name harbor -f new-values.yaml --namespace kube-ops ./harbor
```

### 6. 查看ingress

```shell
kubectl get ingresses. -n kube-ops
NAME                    HOSTS                               ADDRESS                                                                                                 PORTS     AGE
harbor-harbor-ingress   harbor.mytest.io,notary.mytest.io   172.18.1.14,.....,172.18.1.9   80, 443   101m
```

### 7. 本地访问

配置本地/etc/hosts文件

```
kube-ip harbor.mytest.io
```

使用浏览器访问`https://harbor.mytest.io`

账号：`admin`

密码：`Harbor12345`

### 8. docker本地访问配置

使用 docker cli 来进行 pull/push 镜像，由于上面我们安装的时候通过 Ingress 来暴露的 Harbor 的服务，而且强制使用了 https，所以如果我们要在终端中使用我们这里的私有仓库的话，就需要配置上相应的证书：

```
docker login harbor.mytest.io
Username: admin
Password: 
Error response from daemon: Get https://harbor.mytest.io/v2/: x509: certificate signed by unknown authority
```

这是因为我们没有提供证书文件，我们将使用到的ca.crt文件复制到/etc/docker/certs.d/registry.qikqiak.com目录下面，如果该目录不存在，则创建它。ca.crt 这个证书文件我们可以通过 Ingress 中使用的 Secret 资源对象来提供：

```yaml
kubectl get secret harbor-harbor-ingress -n kube-ops -o yaml
apiVersion: v1
data:
  ca.crt: LS0tLS1CRUdJ...
  tls.crt: LS0tLS1CRdJ...
  tls.key: LS0tLS1CRUd...
kind: Secret
metadata:
  creationTimestamp: "2019-07-09T07:30:49Z"
  labels:
    app: harbor
    chart: harbor
    heritage: Tiller
    release: harbor
  name: harbor-harbor-ingress
  namespace: kube-ops
  resourceVersion: "1805594"
  selfLink: /api/v1/namespaces/kube-ops/secrets/harbor-harbor-ingress
  uid: 79634a0e-a21b-11e9-984b-0017fa037437
type: kubernetes.io/tls
```

其中 data 区域中 ca.crt 对应的值就是我们需要证书，不过需要注意还需要做一个 base64 的解码，这样证书配置上以后就可以正常访问了。

```shell
kubectl get secret harbor-harbor-ingress -n kube-ops -o jsonpath="{.data.ca\.crt}"|base64 --decode

-----BEGIN CERTIFICATE-----
MIIC9DCCAdygAwIBAgIQEZIgi3AhzJ8htXYR3fzC+jANBgkqhkiG9w0BAQsFADAU
MRIwEAYDVQQDEwloYXJib3ItY2EwHhcNMTkwNzA5MDc1NTI5WhcNMjAwNzA4MDc1
NTI5WjAUMRIwEAYDVQQDEwloYXJib3ItY2EwggEiMA0GCSqGSIb3DQEBAQUAA4IB
DwAwggEKAoIBAQCyG22aCWdqcsZd39t6O/qbIG/DUGQhC3VxBsTvR6XIXMUu+8vl
WTKk8qMO0bWI8cQwilthJDA50h5POsAUYBBwWNtlE4lR/1FIx2tKSy42Fnd61GVr
ThG6xh4mjp1v5OlerCqCXSVum695xFAi65Pdg2qsw4bCOCeaG5wynDQ3W0+T9pPJ
z8/Xp59rRP+Y3ulTYv5bdY6rczKNGL432SsutyI+TrQvo2NApOFbrZwMg66WTuL2
+n3YPUoQYHzT5RumURowSbrORS7Tuls3fp3rjIrkZ3Z5rY2YHikpkj03vOEdYV0P
7Jxah7gzuVqpSHRzr8yG9aJ+arzsG6BxaTnTAgMBAAGjQjBAMA4GA1UdDwEB/wQE
AwICpDAdBgNVHSUEFjAUBggrBgEFBQcDAQYIKwYBBQUHAwIwDwYDVR0TAQH/BAUw
AwEB/zANBgkqhkiG9w0BAQsFAAOCAQEArBtwin0wswSWA1lCshYszq4+n8acv1id
NMhPzql3iqs+dFWNLAnVYKrT6acOPTOJ2O/pYcpap0/6EMMErSiXpKattjB6b2Sg
+X/ExJYJHust8cgx8rtfjbNWdLgrTIL7tMRQ8dd3NZ/WpiYK3A33wRi4Zjm0JY4e
JhAukw8j6h0QxBaedXhnzNd3i+5w92R/QC616vhjP0TPgeosVnQB02R7fTg5BXwl
vxHazvNMmn3BfReRCXiJrfiqAxzjNN25yyMlnQiGsBMbxbw19Q/sWAy9k/PXgOT5
PJ0pgCZTF0cxXP3d7mI2Lld53z+J6ojqtvg/oyaLzM3m1Af0bC0Kaw==
-----END CERTIFICATE-----
```

手动创建docker下面的harbor专用的证书文件夹

```
sudo mkdir -pv /etc/docker/certs.d/harbor.mytest.io
```

将上面生成的密钥写入到文件夹里面的ca.crt中

```
sudo -i
cat >> /etc/docker/certs.d/harbor.mytest.io/ca.crt<< EOF
-----BEGIN CERTIFICATE-----
MIIC9DCCAdygAwIBAgIQEZIgi3AhzJ8htXYR3fzC+jANBgkqhkiG9w0BAQsFADAU
MRIwEAYDVQQDEwloYXJib3ItY2EwHhcNMTkwNzA5MDc1NTI5WhcNMjAwNzA4MDc1
NTI5WjAUMRIwEAYDVQQDEwloYXJib3ItY2EwggEiMA0GCSqGSIb3DQEBAQUAA4IB
DwAwggEKAoIBAQCyG22aCWdqcsZd39t6O/qbIG/DUGQhC3VxBsTvR6XIXMUu+8vl
WTKk8qMO0bWI8cQwilthJDA50h5POsAUYBBwWNtlE4lR/1FIx2tKSy42Fnd61GVr
ThG6xh4mjp1v5OlerCqCXSVum695xFAi65Pdg2qsw4bCOCeaG5wynDQ3W0+T9pPJ
z8/Xp59rRP+Y3ulTYv5bdY6rczKNGL432SsutyI+TrQvo2NApOFbrZwMg66WTuL2
+n3YPUoQYHzT5RumURowSbrORS7Tuls3fp3rjIrkZ3Z5rY2YHikpkj03vOEdYV0P
7Jxah7gzuVqpSHRzr8yG9aJ+arzsG6BxaTnTAgMBAAGjQjBAMA4GA1UdDwEB/wQE
AwICpDAdBgNVHSUEFjAUBggrBgEFBQcDAQYIKwYBBQUHAwIwDwYDVR0TAQH/BAUw
AwEB/zANBgkqhkiG9w0BAQsFAAOCAQEArBtwin0wswSWA1lCshYszq4+n8acv1id
NMhPzql3iqs+dFWNLAnVYKrT6acOPTOJ2O/pYcpap0/6EMMErSiXpKattjB6b2Sg
+X/ExJYJHust8cgx8rtfjbNWdLgrTIL7tMRQ8dd3NZ/WpiYK3A33wRi4Zjm0JY4e
JhAukw8j6h0QxBaedXhnzNd3i+5w92R/QC616vhjP0TPgeosVnQB02R7fTg5BXwl
vxHazvNMmn3BfReRCXiJrfiqAxzjNN25yyMlnQiGsBMbxbw19Q/sWAy9k/PXgOT5
PJ0pgCZTF0cxXP3d7mI2Lld53z+J6ojqtvg/oyaLzM3m1Af0bC0Kaw==
-----END CERTIFICATE-----
EOF
```

重启docker

```
systemctl restart docker
```

### 9. docker login登录

使用docker-cli登录测试`harbor.mytest.io`

```
docker login harbor.mytest.io
Username: admin
Password: 
WARNING! Your password will be stored unencrypted in /root/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store

Login Succeeded
```

### 10. 上传个镜像试试

拉取个busybox镜像到本地

```
docker pull busybox
Using default tag: latest
latest: Pulling from library/busybox
8e674ad76dce: Pull complete 
Digest: sha256:c94cf1b87ccb80f2e6414ef913c748b105060debda482058d2b8d0fce39f11b9
Status: Downloaded newer image for busybox:latest
```

将busybox的tags修改成`harbor.mytest.io/library/busybox:latest`，`library`是harbor默认的库

```
docker tag busybox:latest harbor.mytest.io/library/busybox:latest
```

使用docker push将修改tag后的镜像推送到harbor

```
docker push harbor.mytest.io/library/busybox:latest
The push refers to repository [harbor.mytest.io/library/busybox]
6194458b07fc: Pushed 
latest: digest: sha256:bf510723d2cd2d4e3f5ce7e93bf1e52c8fd76831995ac3bd3f90ecc866643aff size: 527
```

图图图示：

![](https://note.youdao.com/yws/api/personal/file/WEB84756f145541cc8378afeca4997be935?method=download&shareKey=173eca77c444e7d382f3d2e2f46a18a6)

### 11. 注意

在结合jenkins使用cicd过程中需要注意：

1. 每台node节点需要配置dns解析，否则docker无法访问

2. 需要将刚才上述的密钥推送到全部node节点上，不然会报错，cicd构建过程会失败

3. 可以在pipeline的过程中制定nodeselector，定义其中一台node专门运行jenkins，这台node节点需要打上labels，jenkins通过nodeseletor=labels，可以在一台服务器上报错images了

