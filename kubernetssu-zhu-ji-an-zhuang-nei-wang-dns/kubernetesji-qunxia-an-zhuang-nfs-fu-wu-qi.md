

找一台服务器搭建一台nfs服务器

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



