---
title: Rsyslog部署
date: 2021-07-02 11:05:40
tags: [syslog]
---
# Rsyslog部署

## 简介

syslog是类unix操作系统常用的日志处理程序

rsyslog是Ubuntu系统默认的日志程序，官方的介绍

> RSYSLOG is the **r**ocket-fast **sys**tem for **log** processing.

总体来说，部署分为服务端和客户端两部分

## 配置项含义

### SELECTOR

local0.*是选择器（selector），指定该条配置对哪些日志生效。由类型（facility）和等级（priority）组成

类型包括

```shell
kern     内核信息，首先通过 klogd 传递；
user     用户进程；
mail     邮件；
daemon   后台进程；
authpriv 授权信息；
syslog   系统日志；
lpr      打印信息；
news     新闻组信息；
uucp     由uucp生成的信息
cron     计划和任务信息。
mark     syslog 内部功能用于生成时间戳
local0----local7   自定义程序使用，例如使用 local5 作为 ssh 功能
*        通配符代表除了 mark 以外的所有功能
```

等级包括

```shell
emerg 或 panic   该系统不可用（最紧急消息）
alert            需要立即被修改的条件（紧急消息）
crit             阻止某些工具或子系统功能实现的错误条件（重要消息）
err              阻止工具或某些子系统部分功能实现的错误条件（出错消息）
warning          预警信息（警告消息）
notice           具有重要性的普通条件（普通但重要的消息）
info             提供信息的消息（通知性消息）
debug            不包含函数条件或问题的其他信息（调试级-信息量最多）
none             没有重要级，通常用于排错（不记录任何日志消息）
*                所有级别，除了none
```

可以指定多个选择器，用逗号分隔

### ACTION

紧跟着选择器的是动作（action），表示对所选的日志进行的操作，例如可以存到文件、发送到终端、发送到远程服务器。

此处action使用了rsyslog的高级配置，除了指定IP和端口之外，可以指定重传次数，队列长度

## 客户端部署

### linux部署

#### 安装

目前大多数发行版默认使用rsyslog进行日志记录，未安装时一般使用包管理工具可直接安装
`yum install rsyslog -y`
`apt install -y rsyslog`

主配置文件的位置是/etc/rsyslog.conf，可以将用户配置置于/etc/rsyslog.d文件夹内。

#### 配置

可以创建bga.conf用于传输日志到另一台服务器

```shell
local0.*  action(type="omfwd" target="192.168.1.120" port="514" protocol="tcp"
                 action.resumeRetryCount="10"
                 queue.type="linkedList" queue.size="10000")
# local0.*为筛选器，格式为："日志类型.日志级别"
# target为服务器的IP，protocol为采用的协议类型
```

## 服务器部署

### Linux部署

#### 安装

安装与服务器端安装一致即可

#### 配置

`edit /etc/rsyslog/rsyslog.conf`编辑配置文件：

```shell
module(load="imtcp")  # 载入tcp模块，只能有一条
input(type="imtcp" port="514")  # 启动监听端口，可以有多条，监听多个端口

local0.* /var/log/bga.log  # 将所有类型为local0的日志存储到/var/log/bga.log文件
```

### Windows部署

#### 安装

主流的软件包括KiwiSyslog和winsyslog等软件，其中kiwi有free trail，但现在获取需要2个工作日，很不方便

1. 下载文件[kiwisyslog-with-crack](https://onedrive.live.com/?authkey=%21ABFrSuLKv%5FsDXO4&id=E4C9966B2424DB6F%211704&cid=E4C9966B2424DB6F)
2. 下载完成后解压运行`Kiwi_Syslogd_8.3.7.setup.exe`进行安装
3. 安装完成后进入`crack`目录，选择`Service`目录，打开managerc程序(exe文件)即完成服务的开启

#### 防火墙配置

1. 打开防火墙高级选项，添加入站规则，选择特定端口(端口设置为514或与服务器端口一致)，协议选择udp，并选择允许所有连接；重复上述步骤再添加一条tcp协议的入栈规则

#### syslog server配置

1. 工具栏依次选择manage->Start the Syslogd service
2. 工具栏依次选择File->Setup
   - 在Inputs中选择UDP设置UDP Port为514，Data encoding选择utf-8，Bind to address留空即可
   - 选择TCP设置端口为514，编码同样选择utf-8
3. 依次点击apply->ok

## Debug

python官方库是自带syslog支持的，可以直接用

```python
import syslog

syslog.openlog(ident='BGA', logoption=syslog.LOG_PID, facility=syslog.LOG_LOCAL0)
syslog.syslog("This is a test message.")

# facilities: LOG_KERN, LOG_USER, LOG_MAIL, LOG_DAEMON, LOG_AUTH, LOG_LPR, LOG_NEWS, LOG_UUCP, LOG_CRON, LOG_SYSLOG, LOG_LOCAL0 to LOG_LOCAL7, and, if defined in <syslog.h>, LOG_AUTHPRIV.

# logoptions: LOG_PID, LOG_CONS, LOG_NDELAY, and, if defined in <syslog.h>, LOG_ODELAY, LOG_NOWAIT, and LOG_PERROR.
```

查看服务器的/var/log/bga.log文件，新增了一条日志

```shell
Mar 21 15:22:23 bolean BGA[4295]: This is a test message.
```

日志格式是timestamp hostname ident[pid]：log message
