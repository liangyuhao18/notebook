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
