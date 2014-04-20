---
layout: post
title: "ceph commands (2) - setting debug log level"
date: 2014-04-21 00:04:27 +0800
comments: true
categories: ceph
---

一般情况下，当系统运行一切顺利时，我们无须开启log，这对系统资源是一种浪费。然而有时我们不管是debug还是了解系统运行情况都需要实时的查看修改参数。系统log的设置可以通过多种方式来设置，比如系统配置文件ceph.conf,linux下配置log 速率的logrotate，以及ceph提供的在线查看修改方式。

<!-- more -->

## log配置格式
每个ceph子系统都可以设置log level与memory level，分别表示输出的log和在内存中的log。ceph如果分别来设置这两个值，则需要以斜杠“/”来分开，前面的用来表示log level，后面的表示memory level，如：

    debug ms = 1/5
也可以只指定一个，那这两项就都会使用这个值。到ceph的log级别取值范围为1-20，1表示最少量log，20表示最多。在调试完之后最好将log级别调回默认或系统资源可以承受的级别，否则ceph的log会占用大量的资源。ceph系统默认的log级别可以参照[default log level][1]

## 在配置文件中设置
debug相关的设置是可以通过配置文件ceph.conf来指定的，如：
```
[global]
        debug ms = 1/5

[mon]
        debug mon = 20
        debug paxos = 1/5
        debug auth = 2

[osd]
        debug osd = 1/5
        debug filestore = 1/5
        debug journal = 1
        debug monc = 5/20

[mds]
        debug mds = 1
        debug mds balancer = 1
        debug mds log = 1
        debug mds migrator = 1
```
这比较适合debug ceph启动时的问题，但log会消耗系统资源，在系统没有问题时最好还是关闭，在需要时再打开。
ceph系统可以配置的log选项可见[ceph log options][2]

## 使用logrotate
logrotate是一个管理linux下log文件相关的工具，可以用来压缩，转储，甚至邮寄log。ceph对应的配置文件路径为：`/var/logrotate.d/ceph`，其默认配置如下：
```
/var/log/ceph/*.log {
    rotate 7
    daily
    compress
    sharedscripts
    postrotate
        if which invoke-rc.d > /dev/null 2>&1 && [ -x `which invoke-rc.d` ]; then
            invoke-rc.d ceph reload >/dev/null
        elif which service > /dev/null 2>&1 && [ -x `which service` ]; then
            service ceph reload >/dev/null
        fi
        # Possibly reload twice, but depending on ceph.conf the reload above may be a no-op
        if which initctl > /dev/null 2>&1 && [ -x `which initctl` ]; then
            # upstart reload isn't very helpful here:
            #   https://bugs.launchpad.net/upstart/+bug/1012938
            initctl list \
                | sed -n 's/^\(ceph-\(mon\|osd\|mds\)\+\)[ \t]\+(\([^ \/]\+\)\/\([^ \/]\+\))[ \t]\+start\/.*$/\1 cluster=\3 id=\4/p' \
                | while read l; do
                initctl reload -- $l 2>/dev/null || :
            done
        fi
    endscript
    missingok
}
```
我们可以通过修改其来指定log的一些参数，比如，log文件大小
```
/var/log/ceph/*.log {
    rotate 7
    daily
    size 500M
    compress
    sharedscripts
```
然后使用`crontab -e`来增加如下一行：
    
    30 * * * * /usr/sbin/logrotate /etc/logrotate.d/ceph >/dev/null 2>&1
表示每30分钟检查一下`/etc/logrotate.d/ceph`文件。


## 在线查看log level
使用以下命令dump出指定daemon的所有配置：
    
    # ceph --admin-daemon {/path/to/admin/socket} config show | less
也可以获取具体某项配置的值：
```
# ceph --admin-daemon /var/run/ceph/ceph-mds.sjs_139_137.asok config get mon_lease
{ "mon_lease": "5"}
```
## 在线修改log level
在线修改log level可以有2种方式，`ceph tell`与`ceph --admin-daemon`两种方式，如：

    # ceph tell osd.0 injectargs '--debug-osd 0/5'
这种方式通过monitors，所以凡是可以连接上monitor的都可以执行。
    
    # ceph --admin-daemon /var/run/ceph/ceph-osd.0.asok config set debug_osd 0/5
该命令也可以达到相同的效果，不同是必须在osd.0所处的host上执行。
 

  [1]: http://ceph.com/docs/master/rados/troubleshooting/log-and-debug/#subsystem-log-and-debug-settings
  [2]: http://ceph.com/docs/master/rados/troubleshooting/log-and-debug/#logging-settings
