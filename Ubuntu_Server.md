# Django 部署到 Ubuntu 20.04

---

### 安全的访问

---

进入 Ubuntu 系统后首先设置使用 SSH 免密登录，关闭系统的密码登录。

在你的本地主机上生成公钥、私钥

```shell
ssh-keygen
```

生成的秘钥保存在(本地主机)~/.ssh 文件下面。然后登录到远程主机，在当前用户根目录下使用以下命令创建.ssh 文件夹和.ssh/authorized_keys 文件

```shell
mkdir .ssh
touch .ssh/authorized_keys
```

回到本地主机将公钥上传到远程服务器

```shell
cat .ssh/id_rsa.pub >> 用户名@远程主机IP:~/.ssh/authorized_keys
```

再登录远程主机修改 sshd_config 的配置文件，禁止 root 账户登录和密码登录

```shell
sudo vi /etc/ssh/sshd_config
```

找到 PermitRootLogin，设置为 no，PasswordAuthentication 设置为 no，保存退出。

```shell
PermitRootLogin no
PasswordAuthentication no
```

重启 sshd 服务

```shell
sudo systemctl reload sshd
```

### 简单的防火墙设置

---

查看哪些应用已向防火墙注册

```shell
sudo ufw app list
```

允许 OpenSSH

```shell
sudo ufw allow OpenSSH
```

启用防火墙

```shell
sudo ufw enable
```

检查状态

```shell
sudo ufw status
```

我们现在已经完成了访问和安全性工作，并将继续安装软件

### 安装软件

---

```shell
sudo apt update
sudo apt upgrade
```

安装 pip,git,mysql 和 Nginx

```shell
sudo apt install python3-pip python3-dev mysql-server libmysqlclient-dev nginx git
pip install mysqlclient
```

然后，只需确保我们能够正确安装它即可。 首先，我们将切换到 root 用户。

```shell
sudo su
```

su 命令代表“切换用户”。 如果我们不指定用户，则 Linux / Unix 默认为 root 用户，即管理用户。 该命令提供对系统中所有内容的访问，因此，如果您不熟悉所使用命令的效果，可能会很危险。 注意不要键入以下命令，除非稍后再退出 root 用户。

下面的命令将打开 MySQL 控制台。 您之前没有在命令行中与 MySQL 进行过交互，因为我们改用 MySQL 工作台进行交互。 我们可能没有可用的 GUI，但是在命令行中与 MySQL 交互非常容易。

```shell
mysql -u root -p
```

看到输入密码是直接 Enter，默认空密码。
让我们为我们的项目创建一个数据库。 您可以随意命名，但我们建议您提供项目名称。 注意，每个命令必须以分号结尾。 如果有任何错误或命令未运行，请确保检查它们。 现在在我们的 mysql 服务器上创建一个数据库。

```shell
CREATE DATABASE {{数据库名称}};
```

查看数据库是否创建成功

```shell
SHOW DATABASES;
USE MYSQL;
ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY '你的密码';
```

通过键入 exit 退出 MySQL 提示符；

**重要！！！ 退出 MySQL 提示符后，您必须再次输入命令 exit！**

没错，我们输入了两次 exit。 第一次是退出 MySQL 提示符，第二次是停用 root 用户。 正如我们上面警告的那样，这是至关重要的一步，如果我们跳过它，可能会导致安装出现一些问题。

既然我们已经完成了 MySQL 的所有设置，就可以在 settings.py 文档中更改某些行了，我们可以开始使用 MySQL 数据库了！

如果您在外部项目目录中，则必须 cd 进入包含 settings.py 文件的目录。 如果您按照说明进行操作，则将键入：

```shell
cd {{项目名称}}
sudo vi settings.py
```

更改 settings.py 中的数据库部分，如下所示：

```shell
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.mysql',
        'NAME': '{{数据库名称}}',
        'USER': 'root',
        'PASSWORD': '{{你的密码}}',
        'HOST': 'localhost',
        'PORT': '3306',
    }
}
```

记住如何通过按 esc：wq 退出 vi
现在剩下要做的就是迁移！

```python
python manager.py makemigrations
python manager.py migrate
```

创建管理员账户

```python
python manager.py createsuperuser
```

### Gunicorn 配置

---

打开 gunicorn.socket 文件

```shell
sudo vi /etc/systemd/system/gunicorn.socket
```

复制这段代码，粘贴并保存

```shell
[Unit]
Description=gunicorn socket

[Socket]
ListenStream=/run/gunicorn.sock

[Install]
WantedBy=sockets.target
```

打开 gunicorn.service 文件

```shell
sudo vi /etc/systemd/system/gunicorn.service
```

复制这段代码，粘贴并保存

```shell
[Unit]
Description=gunicorn daemon
Requires=gunicorn.socket
After=network.target

[Service]
User=liuad
Group=liuad
WorkingDirectory=/home/liuad/ReactDjango_guohuaTools/backend
ExecStart=/home/liuad/venv/bin/gunicorn \
          --access-logfile - \
          --workers 3 \
          --bind unix:/run/gunicorn.sock \
          backend.wsgi:application

[Install]
WantedBy=liuad.target
```

启动并启用 gunicorn.socket

```shell
sudo systemctl start gunicorn.socket
sudo systemctl enable gunicorn.socket
```

检查 gunicorn 的状态

```shell
sudo systemctl status gunicorn.socket
```

检查是否存在 gunicorn.sock

```shell
file /run/gunicorn.sock
```

### NGINX 配置

---

创建项目文件夹

```shell
sudo vi /etc/nginx/sites-available/{{项目名称}}
```

复制代码粘贴到文件中

```shell
server {
  listen 80;
  server_name {{IP地址}};

  location = /favicon.ico { access_log off; log_not_found off; }
  location /static/ {
    root /home/liuad/ReactDjango_guohuaTools/backend/frontend/build;
  }

  location /media/ {
    root /home/liuad/ReactDjango_guohuaTools/backend;
  }

  location / {
    include proxy_params;
    proxy_pass http://unix:/run/gunicorn.sock;
  }
}
```

通过链接到启用站点的目录来启用文件

```shell
sudo ln -s /etc/nginx/sites-available/backend /etc/nginx/sites-enabled
```

检查 Nginx 配置

```shell
sudo nginx -t
```

从防火墙中删除端口 8000 并打开防火墙，以允许端口 80 上的正常通信

```shell
sudo ufw delete allow 8000
sudo ufw allow 'Nginx Full'
```

#####您可能需要增加最大上传大小，才能创建包含图片的列表
打开 nginx conf 文件

```shell
sudo vi /etc/nginx/nginx.conf
```

将此添加到 http {}区域

```shell
client_max_body_size 20M;
```

重载 NGINX

```shell
sudo systemctl restart nginx
```

媒体文件问题
您可能会遇到无法显示图像的问题。 我建议删除所有数据并重新开始，并删除“媒体文件夹”中的“照片”文件夹

```shell
sudo rm -rf media/photos
```

### 域设置

---

转到您的域名注册商并创建以下记录

```shell
@ A Record YOUR_IP_ADDRESS
www CNAME example.com
```

转到服务器上的 setting.py 并更改“ ALLOWED_HOSTS”以包含域

```shell
ALLOWED_HOSTS = ['IP_ADDRESS','example.com', 'www.example.com']
```

编辑/etc/nginx/sites-available/backend

```shell
server {
  listen: 80;
  server_name xxx.xxx.xxx.xxx example.com www.example.com;
}
```

重载 NGINX & Gunicorn

```shell
sudo systemctl restart nginx
sudo systemctl restart gunicorn
```
