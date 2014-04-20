---
layout: post
title: "ceph commands (1) - view and modify options online"
date: 2014-04-20 23:50:01 +0800
comments: true
categories: ceph
---

ceph作为一个庞大的分布式系统，为我们提供了很多的选项参数，对于我们优化配置集群很有利。查看和修改在线系统的参数不管是对优化集群，管理生产系统，还是开发debug都有很大的价值。ceph中查看和修改命令对用户都不是很友好，而且需要了解各项参数的含义，但也正是这样限制了初级用户。若想更好的管理集群，需要更加深入的了解系统。

<!-- more -->

在集群配置文件ceph.conf中只保存很有限的参数，而我们不止需要查看其当前值，也需要实时修改。以下是使用ceph-deploy部署的一个测试集群其默认的配置文件：
```
# cat /etc/ceph/ceph.conf
[global]
fsid = 66f09c83-94be-4751-84d8-0d04dfbb8b07
mon_initial_members = sjs_139_137, sjs_131_103, sjs_139_179
mon_host = 10.16.139.137,10.16.131.103,10.16.139.179
auth_cluster_required = cephx
auth_service_required = cephx
auth_client_required = cephx
filestore_xattr_use_omap = true
```

更多的参数，如osd_journal_size， mon_osd_down_out_interval等很多很多的参数都不会展示，本文给出一种在线查看，修改系统参数的方式。
## 在线查看集群参数
在运行的ceph系统中，预留了一个admin socket以供我们操作集群，具体位置为：

    # /var/run/ceph/$cluster-$name.asok
\$cluster与$name为元变量，分别表示集群名与daemon名称，以下为一个socket具体例子：

    # /var/run/ceph/mds.sjs_139_137.pid
想查看该daemon的全部配置参数只需执行如下命令：

    # ceph --admin-daemon /var/run/ceph/$cluster-$name.asok config show | less
admin socket不止支持查询，还有其他的作用，具体可以使用help查询：
```
# ceph --admin-daemon /var/run/ceph/ceph-mds.sjs_139_137.asok help
{ "config get": "config get <field>: get the config value",
  "config set": "config set <field> <val> [<val> ...]: set a config variable",
  "config show": "dump current config settings",
  "get_command_descriptions": "list available commands",
  "git_version": "get git sha1",
  "help": "list available commands",
  "log dump": "dump recent log entries to log file",
  "log flush": "flush log entries to log file",
  "log reopen": "reopen log file",
  "objecter_requests": "show in-progress osd requests",
  "perf dump": "dump perfcounters value",
  "perf schema": "dump perfcounters schema",
  "version": "get ceph version"}
```

## 在线修改集群参数
ceph允许用户在集群运行时修改ceph-osd,ceph-mon,ceph-mds等daemon的参数。这对运行时修改log级别，激活关闭debug设置，还有在线调优有很大的作用。使用以下命令可以轻松修改系统参数，格式如下：

    # ceph {daemon-type} tell {id or *} injectargs '--{name} {value} [--{name} {value}]'
    # ceph tell {daemon-type}.{daemon id or *} injectargs '--{name} {value} [--{name} {value}]'
{daemon-type}表示osd, mon或者mds之中之一，我们可以指定具体daemon的ID，或者使用通配符\*对集群中所有同一类型的daemon做统一修改。如以下命令增加osd.0的debug log级别：
    
    # ceph tell osd.0 injectargs '--debug-osd 0/5'
`ceph tell`命令会经过monitor。我们也可以直接登录到运行指定daemon的host，使用admin socket提供的config set子命令来修改参数：
    
    # ceph --admin-daemon /var/run/ceph/ceph-osd.0.asok config set debug_osd 0/5
    
>NOTE:
	在ceph.conf配置文件中，可能使用空格来分割各个配置，比如`osd journal size = 5120`,但在命令行中使用这些时必须使用-或者_来连接个项，如：`debug-osd`


