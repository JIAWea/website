---
title: 使用Supervisor管理Golang程序
date: 2020-06-20 10:47:23
categories: 后端
tags: [Golang,Supervisor]
---

### 1.介绍
Supervisor是用Python开发的一套通用的进程管理程序，能将一个普通的命令行进程变为后台`daemon`，并监控进程状态，异常退出时能自动重启。

### 2.安装

由于使用Python开发的，所以我们可以用`pip`来安装。
```
sudo pip install supervisor
```

<!--more-->

如果是 Ubuntu 系统，还可以使用 apt-get 安装。
```
apt-get install supervisor
```

### 3.配置
安装完 supervisor 之后，可以运行echo_supervisord_conf 命令输出默认的配置项，也可以重定向到一个配置文件里:
```
echo_supervisord_conf > /etc/supervisord.conf
```

打开配置文件，主要修改最后两行内容，设置配置文件目录并将注释去掉。
```
vim /etc/supervisord.conf
```

```
[unix_http_server]
···
···

; 包含其他的配置文件
[include]
files = /etc/supervisor/*.ini
```

创建放置配置文件的目录
```
mkdir -p /etc/supervisor
```

在目录`/etc/supervisor`下新建配置文件
```
vim /etc/supervisor/demo.ini
```

并添加内容：
```
[program:demo]
directory = /home/demo
command = ./demo.app
autostart = true
startsecs = 5                                                  ; 5 秒后没有异常退出，就当作已经正常启动了
autorestart = true                                             ; 程序异常退出后自动重启
startretries = 3                                               ; 启动失败自动重试次数，默认是 3
user = root                                                    ; 启动用户
redirect_stderr = true                                         ; 把 stderr 重定向到 stdout，默认 false
stdout_logfile_maxbytes = 50MB                                 ; stdout 日志文件大小，默认 50MB
stdout_logfile_backups = 20                                    ; stdout 日志文件备份数
stdout_logfile = /home/demo/log/supervisor/run.log         	   ; 需要事先创建好目录，否则会启动失败
stderr_logfile = /home/demo/log/supervisor/run.err.log     	   ; 需要事先创建好目录，否则会启动失败
```

运行`supervisor`，到这里就部署完成了。
```
supervisord -c /etc/supervisord.conf
```

查看`demo`是否运行成功。
```
supervisorctl status demo
```

### 4.常用命令
```
supervisorctl status        	# 查看所有进程的状态
supervisorctl stop all       	# 停止所有
supervisorctl start all      	#　启动所有
supervisorctl restart       	# 重启
supervisorctl update        	# 配置文件修改后使用该命令加载新的配置
supervisorctl reload        	# 重新启动配置中的所有程序
```
