操作系统：Ubuntu 16.04

服务器IP：172.18.1.12

#### 1. 安装bind9

```
apt install bind9 -y
```

#### 2. 配置解析文件

进入/etc/bind目录下，打开named.conf.local

```shell
//
// Do any local configuration here
//

// Consider adding the 1918 zones here, if they are not used in your
// organization
//include "/etc/bind/zones.rfc1918";
// 添加一下内容
// mytest.io 需要解析的根域名
// file     需要配置解析文件的名称
zone "mytest.io" {
        type master;
file "db.mytest.io";
};
```

#### 3. 配置域解析文件

```shell
cp /etc/bind/db.local /var/cache/bind/db.mytest.io
```

#### 4. 配置域解析文件

```shell
cd /var/cache/bind/

cat db.mytest.io 
; BIND data file for local loopback interface
$TTL    604800
@       IN      SOA      dns.mytest.io. root.mytest.io. (
                              2         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
@       IN      NS      dns.mytest.io.
dns     IN      A       172.18.1.12
rancher IN      A       172.18.1.12
rancher IN      A       172.18.1.13
```

#### 5. 设置外网转发

打开/etc/bind/name.conf.options中添加上forwarders，可以实现外网转发

```shell
forwarders {
     8.8.8.8;
};
```

#### 6. 启动bind服务器

```shell
systemctl start bind9
systemctl enable bind9
```

#### 7. 配置dns解析

打开/etc/resolv.conf文件将dns的IP地址写入

`nameserver 172.18.1.12`

#### 8. 测试dns

```shell
nslookup rancher.mytest.io

Server:         172.18.1.12
Address:        172.18.1.12#53

Name:   rancher.mytest.io
Address: 172.18.1.13
Name:   rancher.mytest.io
Address: 172.18.1.12
```



