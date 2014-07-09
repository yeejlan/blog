title: 'Ubuntu下samba共享服务设置'
date: 2014-07-09 14:23:30
tags: samba
---

Samba实现了windows共享服务，配置完成后可以通过windows网上邻居交换文件。

首先安装samba软件包
```bash
sudo apt-get install samba
```

安装完成后，打开/etc/samba/smb.conf，配置共享目录，个人习惯配置home目录跟/web目录
```config
[homes]
   comment = Home Directories
   browseable = no
   #允许向home目录写入文件
   read only = no

[web]
   path = /web
   guest ok = no
   browseable = yes
   create mask = 0644
   directory mask = 0755
   read only = no
```

配置完成后重启samba服务，使配置文件生效。
```bash
sudo service smbd restart
```

samba有自己的用户授权体系，每个linux系统中原有的用户，都需要重新加入samba的用户库，并且重新设置密码，这样才能访问共享服务。
这里假设加入rose这个用户
```bash
sudo smbpasswd -a rose
```

之后绑定域名dev.example.com到开发机，打开资源管理器，地址栏输入\\dev.example.com就可以访问共享的资源了。
