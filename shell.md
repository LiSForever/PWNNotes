#### iptables端口转发

```shell
# 查看 PREROUTING 链（修改目的地址/端口的规则）
sudo iptables -t nat -L PREROUTING -v -n
# 查看FORWARD链
sudo iptables -L FORWARD -v -n
# 查看 POSTROUTING 链（修改源地址的规则，通常用于 SNAT/MASQUERADE）
sudo iptables -t nat -L POSTROUTING -v -n


# 列出行号
sudo iptables -L FORWARD --line-numbers
sudo iptables -t nat -L PREROUTING --line-numbers
sudo iptables -t nat -L POSTROUTING --line-numbers

# 删除第2条规则
sudo iptables -D FORWARD 3
sudo iptables -t nat -D PREROUTING 2
sudo iptables -t nat -D POSTROUTING 2
```

```shell
# 1. 开启 IP forward
echo 1 > /proc/sys/net/ipv4/ip_forward
sysctl -w net.ipv4.ip_forward=1

# 2. 先清掉所有你加的错误规则（防止重复）
iptables -t nat -D PREROUTING -p tcp --dport 10080 -j DNAT --to-destination 192.168.0.1:80 2>/dev/null
iptables -t nat -D POSTROUTING -p tcp -d 192.168.0.1 --dport 80 -j SNAT --to-source 192.168.108.21:10080 2>/dev/null
iptables -D FORWARD -i ens33 -o tap2_0.1 -p tcp --dport 80 -d 192.168.0.1 -j ACCEPT 2>/dev/null
iptables -D FORWARD -i tap2_0.1 -o ens33 -p tcp --sport 80 -s 192.168.0.1 -j ACCEPT 2>/dev/null

# 3. 正确的 NAT + FORWARD
# 3.1 外部进来的 10080 → 仿真机 80
iptables -t nat -A PREROUTING -p tcp -m tcp --dport 10080 \
         -j DNAT --to-destination 192.168.0.1:80

# 3.2 回包：把仿真机的源地址改成 Ubuntu 的外网 IP（MASQUERADE 自动处理端口）
iptables -t nat -A POSTROUTING -d 192.168.0.1/32 -p tcp --dport 80 \
         -j MASQUERADE

# 3.3 FORWARD 允许双向
iptables -A FORWARD -i ens33 -o tap2_0.1 -d 192.168.0.1 -p tcp --dport 80 -j ACCEPT
iptables -A FORWARD -i tap2_0.1 -o ens33 -s 192.168.0.1 -p tcp --sport 80 -j ACCEPT

# 3.4 （可选）让本机自己也能通过 10080 访问仿真机
iptables -t nat -A OUTPUT -p tcp -d 192.168.108.21 --dport 10080 \
         -j DNAT --to-destination 192.168.0.1:80

# 4. 确保 FORWARD 链默认 ACCEPT（或者至少不 DROP 需要的流量）
iptables -P FORWARD ACCEPT   # 生产环境建议只 ACCEPT 需要的，测试时先放开
```

#### 进程监控

```shell
# 需要使用tee
while true; do ts=$(date '+%Y-%m-%d %H:%M:%S');ps www | sed "s/^/$ts | /" | grep -E 'ldalda|ldalda`|ldalda\$|ldalda&|ldalda\|(sh -c)' | grep -Ev '(grep -E)|gs_keepalived_fifo|(sed s/\^/)' | tee -a /tmp/pid.txt; done

 while true; do ts=$(date '+%Y-%m-%d %H:%M:%S');ps www | sed "s/^/$ts | /" | grep -E 'ldalda|ldalda`|ldalda\$|ldalda&|ldalda\|(sh -c)|/tmp/' | grep -Ev '(grep -E)|gs_keepalived_fi
fo|(/tmp/pid.log)' | awk '{print; print >> "/tmp/pid.log"}'; done
```





