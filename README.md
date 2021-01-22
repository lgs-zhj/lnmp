# lnmp
linux+nginx+mariadb+php-fpm+wordpress


步骤一：安装部署LNMP软件
备注：mariadb（数据库客户端软件）、mariadb-server（数据库服务器软件）、mariadb-devel（其他客户端软件的依赖包）、php（解释器）、php-fpm（进程管理器服务）、php-mysql（PHP的数据库扩展包）。
1）安装软件包
1.	[root@centos7 ~]# yum -y install gcc openssl-devel pcre-devel 
2.	[root@centos7 ~]# useradd -s /sbin/nologin  nginx
3.	[root@centos7 ~]# tar -xvf nginx-1.12.2.tar.gz
4.	[root@centos7 ~]# cd nginx-1.12.2
5.	[root@centos7 nginx-1.12.2]# ./configure   \
6.	--user=nginx   --group=nginx \
7.	--with-http_ssl_module   \
8.	--with-http_stub_status_module
9.	[root@centos7 nginx-1.12.2]# make && make install
10.	
11.	[root@centos7 ~]# yum -y install   mariadb   mariadb-server   mariadb-devel
12.	[root@centos7 ~]# yum -y install   php        php-mysql        php-fpm
2)启动服务(nginx、mariadb、php-fpm)
1.	[root@centos7 ~]# /usr/local/nginx/sbin/nginx                 #启动Nginx服务
2.	[root@centos7 ~]# echo "/usr/local/nginx/sbin/nginx" >> /etc/rc.local
3.	[root@centos7 ~]# chmod +x /etc/rc.local
4.	[root@centos7 ~]# ss -utnlp | grep :80                        #查看端口信息
5.	
6.	[root@centos7 ~]# systemctl start   mariadb                   #启动mariadb服务器
7.	[root@centos7 ~]# systemctl enable  mariadb               
8.	    
9.	[root@centos7 ~]# systemctl start  php-fpm                   #启动php-fpm服务
10.	[root@centos7 ~]# systemctl enable php-fpm
附加知识：systemd！！！
源码安装的软件默认无法使用systemd管理，如果需要使用systemd管理源码安装的软件需要手动编写服务的service文件（编写是可以参考其他服务的模板文件）。以下是nginx服务最终编辑好的模板。
Service文件存储路径为/usr/lib/system/system/目录。
1.	[root@centos7 ~]# vim /usr/lib/systemd/system/nginx.service
2.	[Unit]
3.	Description=The Nginx HTTP Server
4.	#描述信息
5.	After=network.target remote-fs.target nss-lookup.target
6.	#指定启动nginx之前需要其他的其他服务，如network.target等
7.	
8.	[Service]
9.	Type=forking
10.	#Type为服务的类型，仅启动一个主进程的服务为simple，需要启动若干子进程的服务为forking
11.	ExecStart=/usr/local/nginx/sbin/nginx
12.	#设置执行systemctl start nginx后需要启动的具体命令.
13.	ExecReload=/usr/local/nginx/sbin/nginx -s reload
14.	#设置执行systemctl reload nginx后需要执行的具体命令.
15.	ExecStop=/bin/kill -s QUIT ${MAINPID}
16.	#设置执行systemctl stop nginx后需要执行的具体命令.
17.	
18.	[Install]
19.	WantedBy=multi-user.target
3）修改Nginx配置文件，实现动静分离
修改配置文件，通过两个location实现动静分离，一个location匹配动态页面，一个loation匹配其他所有页面。
注意修改默认首页为index.php!
1.	[root@centos7 ~]# vim /usr/local/nginx/conf/nginx.conf 
2.	...省略部分配置文件内容...
3.	location / {
4.	            root   html;
5.	            index  index.php index.html index.htm;           #加上index.php,不然会出现403
6.	        }
7.	...省略部分配置文件内容...
8.	location ~ \.php$ {
9.	            root           html;
10.	            fastcgi_pass   127.0.0.1:9000;
11.	            fastcgi_index  index.php;
12.	            include        fastcgi.conf;
13.	        }
14.	...省略部分配置文件内容...
15.	[root@centos7 ~]# /usr/local/nginx/sbin/nginx -s reload            #重新加载配置
4）配置数据库账户与权限
为网站提前创建一个数据库、添加账户并设置该账户有数据库访问权限。
1.	[root@centos7 ~]# mysql
2.	MariaDB [(none)]> create database wordpress character set utf8mb4;
3.	MariaDB [(none)]> grant all on wordpress.* to wordpress@'localhost' identified by 'wordpress';
4.	MariaDB [(none)]> grant all on wordpress.* to wordpress@'192.168.2.11' identified by 'wordpress';
5.	MariaDB [(none)]> flush privileges;
6.	MariaDB [(none)]> exit
提示：在mysql和mariadb中%代表匹配所有，这里是授权wordpress用户可以从任意主机连接数据库服务器，生产环境建议仅允许特定的若干主机访问数据库服务器。
步骤二：上线wordpress代码
1）上线PHP动态网站代码
1.	[root@centos7 ~]# yum -y install unzip
2.	[root@centos7 ~]# unzip wordpress.zip
3.	[root@centos7 ~]# cd wordpress
4.	[root@centos7 wordpress]# tar -xf wordpress-5.0.3-zh_CN.tar.gz
5.	[root@centos7 wordpress]# cp -r  wordpress/*  /usr/local/nginx/html/
6.	[root@centos7 wordpress]# chown -R apache.apache  /usr/local/nginx/html/
提示：动态网站运行过程中，php脚本需要对网站目录有读写权限，而php-fpm默认启动用户为apache。
2)初始化网站配置（使用客户端访问web服务器IP）
1.	[root@client ~]# firefox http://192.168.2.11/
