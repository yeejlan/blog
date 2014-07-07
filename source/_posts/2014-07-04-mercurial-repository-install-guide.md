title: 'Mercurial仓库架设'
date: 2014-07-04 14:33:30
tags: mercurial
---

Mercurial是使用python编写的，因此需要安装python包管理器，常见的包管理器有easy install和pip。个人推荐使用pip，因为pip比easy install更成熟，不会出现某个python模块安装不完整，导致无法卸载也无法重新安装的问题。

安装easy install
```bash
sudo apt-get install python-setuptools
```

用easy install安装pip 
```bash
sudo easy_install pip
```

在安装mercurial之前, 需要先安装gcc等编译工具，以及python的开发支持库

```bash
sudo apt-get install build-essential python-dev
```
 
安装mercurial
```bash
sudo pip install mercurial
```

到目前为止mercurial已经安装完成了，命令行下输入"hg"就可以看到mercurial的help信息了。接下来配置一个mercurial的公共仓库，方便代码共享。

平时我的开发网站放在/web目录下面，比如域名为demo.example.com的网站，物理路径是/web/demo.example.com

首先初始化一个mercurial的仓库

```bash
mkdir /web/demo.example.com && cd /web/demo.example.com && hg init
```

接下来使用hgweb的wsgi脚本配置一个服务，详细信息可以参考: [http://mercurial.selenic.com/wiki/PublishingRepositories#hgweb](http://mercurial.selenic.com/wiki/PublishingRepositories#hgweb)

个人喜欢gunicorn作为python app的WSGI服务器，因此先安装gunicorn

```bash
sudo pip install gunicorn
```

/web/hgweb.py (wsgi脚本)

```python
# An example WSGI for use with mod_wsgi, edit as necessary
# See http://mercurial.selenic.com/wiki/modwsgi for more information

# Path to repo or hgweb config to serve (see 'hg help hgweb')
config = "/web/hgweb.config"

# Uncomment and adjust if Mercurial is not installed system-wide
# (consult "installed modules" path from 'hg debuginstall'):
#import sys; sys.path.insert(0, "/path/to/python/lib")

# Uncomment to send python tracebacks to the browser if an error occurs:
import cgitb; cgitb.enable()

# enable demandloading to reduce startup time
from mercurial import demandimport; demandimport.enable()

from mercurial.hgweb import hgweb
application = hgweb(config)
```

/web/hgweb.config 配置文件

```config
[paths]
/ = /web/*

[web]
style = monoblue
push_ssl = false
allow_push = *
```

绑定hgrepos.example.com到开发环境IP
gunicorn --workers=2 hgweb:application --bind=0.0.0.0:8001  启动guincorn服务器看看

```
Name			Description		Contact		Last modified	 	 
demo.example.com	unknown			unknown		Fri, 04 Jul 2014 08:12:42 -0400	
```
可以看到前面建立的仓库demo.example.com，成功的列出了。

下面使用apache代理gunicorn的请求，首先启用代理模块

```config
LoadModule proxy_module /usr/lib/apache2/modules/mod_proxy.so
LoadModule proxy_http_module /usr/lib/apache2/modules/mod_proxy_http.so
```

配置apache虚拟主机  hgrepos.example.com.conf (apache2.4需要 "Require all granted", 详细信息见http://httpd.apache.org/docs/2.4/upgrading.html)
```config
<VirtualHost *:80>
ServerName hgrepos.example.com
ProxyRequests On
<Proxy *>
Order deny,allow
Allow from all
Require all granted
</Proxy>
ProxyPass / http://127.0.0.1:8001/
ProxyPassReverse / http://127.0.0.1:8001/
</VirtualHost>
```

如果用nginx，参考配置如下
```config
server {
    listen          80;
    server_name     hgrepos.example.com;
    server_name_in_redirect off;

    access_log  /var/log/nginx/hg_access.log;
    error_log   /var/log/nginx/hg_error.log;
    send_timeout     3m;
    location / {
        proxy_pass http://127.0.0.1:8001;
        proxy_redirect off;
        proxy_connect_timeout 3000;
        proxy_read_timeout 3000;
        proxy_send_timeout 3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        client_body_buffer_size 128k;
        client_max_body_size 2000m;
        proxy_buffering on;
        proxy_buffer_size 4k;
        proxy_buffers 4 32k;
        proxy_busy_buffers_size 64k;
        proxy_temp_file_write_size 64k;
    }
}
```
