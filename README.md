# macvlan-docker-host-access
### 问题产生原因
由于macvlan的安全特性，禁止父网卡与在其下创建的macvlan接口通信

### 解决方式
在宿主机上创建macvlan接口负责与网络模式为macvlan的docker容器通信(相当于同一个父网卡下不同macvlan之间通信)，同时为了解决宿主机中创建好的macvlan接口重启后失效的问题，通过systemd实现每次宿主机重启时执行脚本创建macvlan接口

### 操作步骤
#### 新建一个空文件(文件名随意)
```bash
cd /etc/systemd/system
sudo vim set-macvlan-for-host-container-access.service
```

#### 在新建的文件中填入以下代码
```bash
[Unit]
Description=Configure a macvlan Interface to Enable Host and Docker Container Communication
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
RemainAfterExit=yes
Environment=NIC_NAME=kvmbr0
Environment=MACVLAN_INTERFACE_NAME=nic-macvlan
Environment=IP_ADDRESS=192.168.124.190/27
ExecStart=/bin/bash -c '\
    # 打开物理网卡混杂模式\
    /sbin/ip link set ${NIC_NAME} promisc on;\
    # 创建macvlan网络接口\
    /sbin/ip link add ${MACVLAN_INTERFACE_NAME} link ${NIC_NAME} type macvlan mode bridge;\
    # 为macvlan网络接口设置ip子网地址\
    # 这里设置的子网范围是192.168.124.160/27(网络地址192.168.124.60,广播地址:192.168.124.191,可用地址范围:192.168.124.61~192.168.124.190)\
    # 为创建的macvlan接口赋予IP,此处使用上述规划的子网中最后一个可用IP:192.168.124.190\
    /sbin/ip addr add ${IP_ADDRESS} dev ${MACVLAN_INTERFACE_NAME};\
    # 开启接口\
    /sbin/ip link set ${MACVLAN_INTERFACE_NAME} up;\
'

[Install]
WantedBy=multi-user.target
```

上面的脚本只需要按实际情况修改**Environment**变量的值即可,其他地方保持不变

**NIC_NAME** :  修改成你实际物理网卡的名称(如eth0,kvmbr0等)

**MACVLAN_INTERFACE_NAME **: 最终宿主机上创建出来的macvlan接口的名字,这个名字可随意填写

**IP_ADDRESS **: 你希望哪些ip地址段流量通过该接口，这里最好与你规划给docker的macvlan网络ip地址段相同，毕竟新创建的macvlan接口就是为了负责与docker macvlan通信的.如果不清楚建议不修改，此处模板中规划的子网为 `192.168.124.60/27` (网络地址192.168.124.60，广播地址:192.168.124.191，可用地址范围:192.168.124.61~192.168.124.190),，此处填写192.168.124.190/27意为将子网中的192.168.124.190分配给macvlan接口, 同时划分子网范围

#### 使用systemd实现开机启动
设置可执行权限

```bash
sudo chmod a+x set-macvlan-for-host-container-access.service
```

重新加载systemd服务

```bash
sudo systemctl daemon-reload
```

启用服务使其开机自启

```bash
sudo systemctl enable set-macvlan-for-host-container-access.service
```

立即启用脚本

```bash
sudo systemctl start set-macvlan-for-host-container-access.service
```

#### 查看宿主机上macvlan接口是否创建成功
```bash
ip link
```

在输出中能看到名为nic-macvlan(这个是自己设置的,如果跟我一样才是这个名字)的网卡即表示成功

```bash
ip route
```



### 其他

