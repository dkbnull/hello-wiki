```shell
#转发开启或关闭网络
cat /proc/sys/net/ipv4/ip_forward

#设为开启
echo 1 > /proc/sys/net/ipv4/ip_forward

#永久开启
#修改 /etc/sysctl.conf 文件 把net.ipv4.ip_forward=1写进去 重启网络之后docker网络也正常
echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
```
