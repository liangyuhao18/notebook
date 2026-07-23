#### 关闭挂起，休眠，睡眠
**方法一：通过systemd禁用(推荐)**
禁用：
```
sudo systemctl mask sleep.target suspend.target hibernate.target hybrid-sleep.target
```
重新启用：
```
sudo systemctl unmask sleep.target suspend.target hibernate.target hybrid-sleep.target
```
**方法二：修改配置文件**
编辑配置文件 /etc/systemd/logind.conf
```
sudo nano /etc/systemd/logind.conf
或
sudo vim /etc/systemd/logind.conf 
```
找到并修改以下行
```
HandleSuspendKey=ignore
HandleHibernateKey=ignore
HandleLidSwitch=ignore
HandleLidSwitchDocked=ignore
```
设置完之后重启一下
```
sudo systemctl restart systemd-logind
sudo reboot
```

#### 方法三：通过Grub禁用（不建议，涉及到内核引导文件，很容易变砖）
**编辑Grub配置文件 /etc/default/grub**
```
sudo nano /etc/default/grub
或
sudo vim /etc/default/grub
```
**找到GRUB_CMDLINE_LINUX_DEFAULT行，添加参数**
```
systemd.sleep=suspend-none
```
例如：
```
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash systemd.sleep=suspend-none"
```
保存并退出后，更行Grub配置
```
sudo update-grub
```
重启电脑

#### 应用程序图标显示异常问题
1. 先前往/usr/share/applications下面检查是否有对应应用的.desktop文件。若没有，创建一个。
2. 右键使用文本编辑器打开.desktop文件，会有如下内容
	