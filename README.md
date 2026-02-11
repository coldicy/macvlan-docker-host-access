## 解决macvlan模式下的docker容器无法与宿主机通信问题
解决方法参考了项目[sarunas-zilinskas/docker-compose-macvlan](https://github.com/sarunas-zilinskas/docker-compose-macvlan)，感谢
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
Environment=NIC_NAME=eth0
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

**MACVLAN_INTERFACE_NAME**: 最终宿主机上创建出来的macvlan接口的名字,这个名字可随意填写

**IP_ADDRESS**: 你希望哪些ip地址段流量通过该接口，这里最好与你规划给docker的macvlan网络ip地址段相同，毕竟新创建的macvlan接口就是为了负责与docker macvlan通信的.如果不清楚建议不修改，此处模板中规划的子网为 `192.168.124.60/27` (网络地址192.168.124.60，广播地址:192.168.124.191，可用地址范围:192.168.124.61~192.168.124.190),，此处填写192.168.124.190/27意为将子网中的192.168.124.190分配给macvlan接口, 同时划分子网范围

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
$ ip link
......
5: eth0: <BROADCAST,MULTICAST,PROMISC,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether 24:77:ad:2c:86:a0 brd ff:ff:ff:ff:ff:ff
......
8: nic-macvlan@eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1499 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether 7e:7a:b6:5d:69:b3 brd ff:ff:ff:ff:ff:ff
......
```

使用 `ip link` 命令查看网络接口，在输出中能看到名为nic-macvlan(这个是自己设置的,如果跟我一样才是这个名字)的网卡即表示创建成功

```bash
$ ip route
......
192.168.124.0/24 dev eth0 proto kernel scope link src 192.168.124.30 metric 100 
192.168.124.160/27 dev nic-macvlan proto kernel scope link src 192.168.124.190
......
```
使用 `ip route` 命令查看所有路由，示例中可以看到宿主机创建macvlan网卡后，目的网络位于192.168.124.160/27范围时，数据包将通过`nic-macvlan`网卡进行处理


补充解释一下，ip route输出中可以看到:
父网卡 `eth0` 可以处理目标地址为192.168.124.0/24的数据包
通过父网卡创建的macvlan虚拟网卡 `nic-macvlan` 可以处理目标地址为192.168.124.160/27的数据包
当我们访问192.168.124.170时，从路由表可知eth0和nic-macvlan都可以处理，但根据路由的最长前缀匹配原则，192.168.124.160/27的网络掩码长度为27，比192.168.124.0/24的网络掩码长度24更长，从而选择nic-macvlan网卡进行数据包发送


### 其他
提供了本文所示的脚本及对应的docker-compose供参考
#### 可能存在的问题
父接口（eth0）状态变化： macvlan 依赖于 eth0。如果 eth0 因为 DHCP 续租、网络闪断或 NetworkManager 重新扫描而导致状态瞬间变为 DOWN，内核会立即清除所有依赖于该接口的路由和子接口。一旦发生此种现象宿主机将不再能通过创建的子接口与macvlan模式下的docker容器进行通信（因为路由不在了）
以下方式能解决由于路由消失导致此方法失效的问题


**查看当前路由表中是否还存在 nic-macvlan 的路由信息**
```bash
ip route | grep nic-macvlan
```
如果没有该网卡的输出，证明此路由已被清除。

#### 解决方案
##### 方案一、手动添加路由（临时解决）
```bash
sudo /sbin/ip route add 192.168.124.160/27 dev nic-macvlan
```
##### ~~方案二、借助 NetworkManager 的 Dispatcher 机制~~
###### 创建监听脚本
```bash
sudo vim /etc/NetworkManager/dispatcher.d/fix-nic-macvlan-route
```
粘贴以下逻辑（它会在 eth0 变 UP 的瞬间强行恢复路由）：

```bash
#!/bin/bash

# 参数说明：$1 是接口名（eth0）, $2 是状态（up/down）
IFACE=$1
ACTION=$2

if [ "$IFACE" = "eth0" ] && [ "$ACTION" = "up" ]; then
    # 延迟 2 秒确保协议栈准备好
    sleep 2
    # 补回路由
    /sbin/ip route add 192.168.124.160/27 dev nic-macvlan 2>/dev/null
fi
```
###### 赋予执行权限
```bash
sudo chmod +x /etc/NetworkManager/dispatcher.d/fix-nic-macvlan-route
```
###### 手动执行脚本
本次执行只是为了手动触发脚本恢复路由，因为脚本触发条件是父网卡状态由DOWN恢复成UP的时刻， 此时肯定不满足脚本条件，所以要手动执行一次脚本，后续满足条件脚本会自动执行
```bash
sudo /etc/NetworkManager/dispatcher.d/fix-nic-macvlan-route eth0 up
```
##### 方案三、创建守护脚本监听路由，消失后自动修补
###### 创建守护脚本（名字随意）
```bash
cd /etc/systemd/system
sudo vim fix-macvlan-route.service
```
###### 脚本填入以下内容（Environment根据自己情况填写）
```bash
[Unit]
Description=Service for monitoring and restoring missing Macvlan routes.
After=set-macvlan-for-host-container-access.service

[Service]
Type=simple
Environment=MACVLAN_IF=nic-macvlan
Environment=ROUTE_NET=192.168.124.160/27
ExecStart=/bin/bash -c '\
    echo "Macvlan watchdog started..."; \
    while true; do \
        # 1. 检查网卡是否存在（静默检查，报错才显示）
        if /sbin/ip link show "${MACVLAN_IF}" >/dev/null; then \
            \
            # 2. 检查路由是否存在（通过 grep 判断，不输出内容）
            if ! /sbin/ip route show dev "${MACVLAN_IF}" | /bin/grep -q "${ROUTE_NET}"; then \
                \
                # 3. 只有路由丢失时才尝试添加，并打印一条清晰的日志记录
                echo "Route missing! Attempting to restore ${ROUTE_NET}..."; \
                if /sbin/ip route add "${ROUTE_NET}" dev "${MACVLAN_IF}" 2>&1; then \
                    echo "Route successfully restored."; \
                else \
                    echo "Failed to restore route! See error above."; \
                fi; \
            fi; \
        fi; \
        sleep 5; \
    done'
# 故障自愈：如果 Bash 进程意外停止，自动重启
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```
###### 使用systemd实现开机启动
设置可执行权限

```bash
sudo chmod a+x fix-macvlan-route.service
```

重新加载systemd服务

```bash
sudo systemctl daemon-reload
```

启用服务使其开机自启

```bash
sudo systemctl enable fix-macvlan-route.service
```

立即启用脚本

```bash
sudo systemctl start fix-macvlan-route.service
```
查看该脚本日志
```bash
sudo journalctl -u fix-macvlan-route.service -f
```
