# discuz-linux-notes
## Debian搭建Discuz论坛

适用前提：裸搭discuz

搭建LAMP架构

apt install -y php

安装完成后重启apache:

systemctl restart apache2

接着安装mariadb:

apt install -y mariadb-server

安装完成后会自动启动，这里要注意apache服务器和mariadb服务器是否在同一台服务器上，默认mariadb启用的是127.0.0.1:3306也就是本地的服务，如果需要和apache一起使用就去修改监听端口

vim /etc/mysql/mariadb.conf.d/50-server.cnf

修改bind-addrss的地址修改为0.0.0.0让外部主机都能访问到mariadb服务


下载discuz论坛软件包到服务器本地（只需要软件包中的upload文件）

创建网站目录

mkdir /www/var/html

cp -ar /upload/* /www/var/html


修改 Apache 的 000-default.conf 文件

sudo nano /etc/apache2/sites-available/000-default.conf

<VirtualHost *:80>
    ServerAdmin webmaster@localhost
    DocumentRoot /var/www/html
    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>


后续去浏览器中安装

修改目录属主或者是修改文件的权限

chown webuser.webuser /webdata/discuz -R

chmod 777 ** 这个需要一个一个文件去修改很麻烦，修改属主会快很多

安装依赖包

apt install -y php-mysqli

systemctl restart apache2

apt install -y php-xml

systemctl restart apache2

浏览器下一步

## acme.sh 强制刷新脚本，傻逼宝塔滚蛋

### 安装 acme.sh


```bash
curl  https://get.acme.sh | sh
source ~/.bashrc
acme.sh --version
```

注册 Let’s Encrypt 账户：
```
acme.sh --register-account -m my@example.com --server letsencrypt
```

设置 Let’s Encrypt 为默认 CA
```
acme.sh --set-default-ca --server letsencrypt
```
HTTP验证
```
acme.sh --issue \
  -d xxx.club \
  -w /www/wwwroot/server \
  --server letsencrypt
```

安装证书

acme.sh 默认把证书放在 ~/.acme.sh/域名/ 下。如果你需要在系统中统一管理，比如放到 /etc/ssl/xxx.com/，可使用 --install-cert 命令来自动部署。
自动部署和重载
```
acme.sh --install-cert -d renheng99.club \
  --key-file       /etc/ssl/renheng99.club/privkey.pem \
  --fullchain-file /etc/ssl/renheng99.club/fullchain.pem \
  --reloadcmd      "service nginx reload" \
  --server         letsencrypt
```

然后在你的 Nginx 配置（示例 /www/server/panel/vhost/nginx/xxx.club.conf）写：

server {
    listen 443 ssl http2;
    server_name xxx.club;

    ssl_certificate      /etc/ssl/xxx.club/fullchain.pem;
    ssl_certificate_key  /etc/ssl/xxx.club/privkey.pem;

    root /www/wwwroot/server;
    index index.html index.php;
    # ...
    
}

保存后：
```
nginx -t
nginx -s reload
```

即可生效。
### 配置自动续签 (Cron)
1. 默认自动安装的定时任务

acme.sh 安装后会自动在 crontab -l 中生成一条每天运行的任务：
```
0 0 * * *  "/root/.acme.sh"/acme.sh --cron --home "/root/.acme.sh" > /dev/null
```


## 穷鬼必备：服务器每日定时备份
## crontab+rsync全自动脚本
鉴于都研究了这么久了，万一以后服务器出现不测需要从头再来，把服务器之间的定时备份方法整理成了文档。（啊啊啊啊！我现在好后悔之前的东西都没整理过！！！万一服务器寄了怎么办！）

环境：Linux Debian11 服务器*2（源服务器/备份目标服务器）

前提：想省钱不使用腾讯阿里oss服务/走服务器回国但不想实名


1. 安装 rsync和crontab 工具
```
sudo apt-get update
sudo apt-get install rsync
sudo apt update
sudo apt install cron
```

2. 设置 SSH 免密码登录
```
ssh-keygen -t rsa #在源服务器上生成SSH密钥
```
Generating public/private rsa key pair.

Enter file in which to save the key (/root/.ssh/id_rsa):文件名

Enter passphrase (empty for no passphrase):不想要可以直接enter跳过


3. 复制公钥到备份服务器
```
ssh-copy-id user@target-server
```
一般这个时候就可以直接用源服务器免密码登录备份服务器了
验证方式：
```
ssh -i /root/.ssh/id_rsa/文件名 user@target-server
```
如需详细输出
```
ssh -v -i /root/.ssh/id_rsa/文件名 user@target-server
```
确保密钥文件 chmod 权限至少是600


4.编写 rsync 备份脚本
接下来，可以创建一个脚本来运行rsync命令。假设要备份/home/user/data目录到目标服务器的/backup/data目录：
```
nano /path/to/backup.sh
```
```
#!/bin/bash
rsync -avz --delete /home/user/data user@target-server:/backup/data
```
- -a 表示归档模式，保留原始文件的权限和属性。
- -v 表示详细模式，显示更多信息。
- -z 表示压缩，减少数据传输量。
- --delete 表示删除目的地有但源目录没有的文件。

```
chmod +x backup.sh
```
也可以选择跳过创建脚本这一步，直接把命令写到crontlab里
例：
如备份脚本为sql
```
#mysqldump -uusername -ppassword discuz > .sql
```

5. 设置crontab任务
在源服务器上：
```
crontab -e
```

在crontab文件里添加新一行来定时运行备份脚本
```
0 1 * * * /path/to/backup.sh
```
0 1 * * * 表示每天的1:00（服务器时间）
/path/to/backup.sh 是你的备份脚本的完整路径

在备份服务器上:
```
crontab -e
```
```
0 1 * * * /usr/bin/find /backup/location -type f -name "*.sql" -mtime +5 -exec rm {} \;
```
- 0 1 * * *：cron时间表达式，指示任务每天凌晨1点执行。
- /usr/bin/find：find命令的完整路径，使用which find可以查找到正确的路径。
- /backup/location：您的备份文件所在目录。
- -type f：指示find命令仅查找文件类型为普通文件。
- -name "*.sql"：匹配所有以.sql结尾的文件。
- -mtime +5：匹配所有最后修改时间在5天前的文件。
- -exec rm {} \;：对匹配到的每个文件执行rm命令来删除它。

重新启动crond:
```
#/etc/rc.d/init.d/crond restart
````


6. 一些可能存在的愚蠢错误：

在2.中没有把backup.pub生成在正确的默认文件夹，就需要让服务器知道这个文件在哪里
```
find /root -name "backup*"
ssh-copy-id -i ~/.ssh/path/to/your/.pub user@target-server
eval `ssh-agent -s`
ssh-add /path/to/your/.pub
rsync -avz -e "ssh -i /path/to/your/.pub" /home/data/discuz_$(date +%Y-%m-%d).sql user@target-server:/backup
```

7. 一些可能有用的命令(discuz)

-备份数据
#mysqldump -u root -p ultrax > /home/data/backup-05-09.sql
需要输入密码！

这样就把discuz数据库所有的表结构和数据备份到discuz_2010-04-01.sql里了，
如果数据量大会占用很大空间，这时可以利用gzip压缩数据，
命令如下：
#mysqldump -uusername -ppassword discuz | gzip > discuz_2010-04-01.sql.gz

可手动创建备份：
/usr/bin/mysqldump -uroot -p'YourPasswordHere' discuz > /path/to/backup/discuz_$(date +%Y-%m-%d).sql

系统崩溃，重建系统时，可以这样恢复数据：
#mysql -uusername -ppassword discuz < discuz_2010-04-01.sql

从压缩文件直接恢复：
#gzip < discuz_2010-04-01.sql.gz | mysql -uusername -ppassword discuz

cron任务日志：
grep CRON /var/log/syslog








