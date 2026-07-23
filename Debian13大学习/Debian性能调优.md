**==性能调优涉及到修改内核文件，要小心一点，别乱搞。否则就是重装系统。笔记作者在两个星期内重装四次了==**

### 内存与交换调优
***最好的内存调优方式就是禁用交换空间(swap)，尤其是运行内存较大的设备***
**甚至不建议开启交换空间**
如果启动了交换空间，则可以调整  `swappiness`项[^1]，如下
```
# 查看内存使用
free -h

# 查看交换空间
swapon --show

# 调整 swappiness（推荐值：10-20）
echo 'vm.swappiness=10' | sudo tee -a /etc/sysctl.conf

# 立即应用
sudo sysctl vm.swappiness=10

# 查看swappiness的值
cat /proc/sys/vm/swappiness
或
sysctl -q vm.swappiness
```

[^1]: 在linux中，可以通过修改swappiness内核参数，降低系统对swap的使用，从而提高系统的性能。简单地说这个参数定义了系统对swap的使用倾向，默认值为60，值越大表示越倾向于使用swap。

### 内核与启动项调优
#### 禁用服务

#### 优化内核启动参数（GRUB）
通过调整 GRUB 启动参数，可减少启动时间和资源消耗。**（要小心，很容易导致炸机变砖）**

### 存储性能调优
若使用 SSD，启用 TRIM(回收空闲块)可延长寿命并保持速度：（强烈建议）
```
sudo systemctl enable --now fstrim.timer # 自动定期 TRIM（每周一次）
```
验证：（手动触发一次 TRIM，显示已回收空间）
```
sudo fstrim -v /
```