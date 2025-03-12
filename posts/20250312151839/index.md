# Windows 宿主机ssh连接VMware虚拟机

&gt;

&lt;!--more--&gt;
## 网络适配器

网络连接选择NAT模式

![image.png](https://img-1313049298.cos.ap-shanghai.myqcloud.com/note-img/202503121518404.png)

## 获取宿主机和虚拟机的ip地址

### 宿主机：

在cmd输入命令：
```cmd
ipconfig
```

![image.png](https://img-1313049298.cos.ap-shanghai.myqcloud.com/note-img/202503121518406.png)

此处物理机正在使用的无线网络，所以IP地址是10.151.*** . ***

### 虚拟机：

在虚拟机终端输入：
```bash
ifconfig
```

![image.png](https://img-1313049298.cos.ap-shanghai.myqcloud.com/note-img/202503121518407.png)

此处虚拟机IP地址为：192.168.131.130

**注意：**`ifconfig`如果提示不存在按提示安装即可

```bash
yum install net-tools #centos
apt install net-tools #Debian
```

## 虚拟机和宿主机IP映射

编辑--&gt;虚拟网络编辑器:

选择VMnet8,更改设置

![image.png](https://img-1313049298.cos.ap-shanghai.myqcloud.com/note-img/202503121518408.png)

选择 VMnet8，点击 NAT 设置

![image.png](https://img-1313049298.cos.ap-shanghai.myqcloud.com/note-img/202503121518409.png)

点击添加

![image.png](https://img-1313049298.cos.ap-shanghai.myqcloud.com/note-img/202503121518410.png)

将虚拟机的IP地址和端口号填入(端口默认22)

![image.png](https://img-1313049298.cos.ap-shanghai.myqcloud.com/note-img/202503121518411.png)

确定保存

## 在虚拟机安装ssh服务

openssh-client：
```bash
sudo apt-get install openssh-client
```

openssh-server：
```bash
sudo apt-get install openssh-server
```

 启动 ssh-server：
 ```bash
sudo /etc/init.d/ssh restart
 ```
 
确认 ssh-server 工作正常：
```bash
netstat -tpl # 看到 ssh 表示工作正常
```

## 设置服务器允许root用户ssh登录

完成以上步骤之后仍有可能连接不上服务器,这是因为有些系统不允许ssh使用root用户登录

### 修改/etc/ssh/sshd_config文件

```
vim /etc/ssh/sshd_config
```

设置以下两项为yes
```vim
 PermitRootLogin yes
 PasswordAuthentication yes
```

完成后重启服务器或者ssh即可

---

> 作者:   
> URL: https://lucazou.github.io/posts/20250312151839/  

