## VMware 设置桥接模式

1. 关闭虚拟机里的系统
2. 编辑 -> 虚拟网络编辑器

![image-20191229140019905](images/image-20191229140019905.png)

3. 添加“桥接模式”

   ![image-20191229160233195](images/image-20191229160233195.png)

   ![image-20191229160448542](images/image-20191229160448542.png)

   ![image-20191229160632334](images/image-20191229160632334.png)

4. 虚拟机设置

   ![image-20191229160834682](images/image-20191229160834682.png)

5. 启动虚拟机

   ```shell
   # 查看网卡名称
   ifconfig
   # 修改对应网卡，ens33 为 ifconfig 命令输出的网卡名称
   vi /etc/sysconfig/network-scripts/ifcfg-ens33
   ```

   ![image-20191229162110712](images/image-20191229162110712.png)

将 ONBOOT 修改为yes

保存退出后重启网络服务 `systemctl restart network.service`





