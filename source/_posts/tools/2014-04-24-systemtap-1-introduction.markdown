---
layout: post
title: "systemtap (1) - introduction"
date: 2014-04-24 22:26:43 +0800
comments: true
categories: tools kernel
---
# 开始

## 简介

systemtap是一个在线调试的瑞士军刀，可以为用户展示系统的很多细节，做性能分析,在线debug等。
很多时候它被用来和solaris的Dtrace相提并论。看下官方介绍：
>SystemTap is a tracing and probing tool that allows users to study and monitor the activities of the computer system (particularly, the kernel) in fine detail. It provides information similar to the output of tools like netstat, ps, top, and iostat; however, SystemTap is designed to provide more filtering and analysis options for collected information.

<!-- more -->

## 安装

要使用systemtap,除了必要的systemp包之外，还必须安装相应的debuginfo。

### 安装systemptap

systemtap需要2个包，分别为systemtap,systemtap-runtime。如果使用yum包管理，使用如下命令：

	yum install systemtap systemtap-runtime

### 安装debuginfo

#### 发行版自带

一般发行版都会自带运行内核对应的debuginfo，针对rhel需要安装的包如下：

	yun install kernel-debuginfo kernel-debuginfo-common kernel-devel

#### 手动build内核

内核开发很多时候需要手动编译upstream内核，此时需要使用-r 选项指定相关的目录.
如果-r制定一个目录，则stap会搜索目录中的debuginfo，如：

	stap -r /path/to/kernel/build/tree [...]

否则,stap会搜索/lib/modules/${RELEASE}/build/下的文件，如：

	% stap -r RELEASE [...]

### 验证安装是否成功

使用如下命令查看是否安装成功:

	stap -v -e 'probe vfs.read {printf("Hello World\n"); exit()}'

如果有如下类似输出，则说明安装成功。

	Pass 1: parsed user script and 45 library script(s) in 340usr/0sys/358real ms.
	Pass 2: analyzed script: 1 probe(s), 1 function(s), 0 embed(s), 0 global(s) in 290usr/260sys/568real ms.
	Pass 3: translated to C into "/tmp/stapiArgLX/stap_e5886fa50499994e6a87aacdc43cd392_399.c" in 490usr/430sys/938real ms.
	Pass 4: compiled C into "stap_e5886fa50499994e6a87aacdc43cd392_399.ko" in 3310usr/430sys/3714real ms.
	Pass 5: starting run.
	read performed
	Pass 5: run completed in 10usr/40sys/73real ms.

# 原理

SystemTap内部工作步骤如下：

* SystemTap checks the script against the existing tapset library (normally in /usr/share/systemtap/tapset/ for any tapsets used. SystemTap will then substitute any located tapsets with their corresponding definitions in the tapset library.
* SystemTap then translates the script to C, running the system C compiler to create a kernel module from it. The tools that perform this step are contained in the systemtap package.
* SystemTap loads the module, then enables all the probes (events and handlers) in the script. The staprun in the systemtap-runtime package provides this functionality.
* As the events occur, their corresponding handlers are executed.
* Once the SystemTap session is terminated, the probes are disabled, and the kernel module is unloaded.

# runtime 执行方式

systemTap脚本是可以编译成模块，然后直接运行的.

### 编译systemtap脚本

编译systemp脚本格式如下：

	stap -r kernel_version script -m module_name
	stap -r 2.6.18-92.1.10.el5 -e 'probe vfs.read {exit()}' -m simple

### 运行编译后的模块

使用staprun执行模块：
	
	staprun module_name.ko

