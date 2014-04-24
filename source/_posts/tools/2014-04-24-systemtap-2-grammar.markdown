---
layout: post
title: "systemtap (2) - grammar"
date: 2014-04-24 22:36:12 +0800
comments: true
categories: tools kernel
---
# 语法

SystemTap的语法与C,bash,awk有很多相似之处。其脚本主要有两部分构成，probe event，
probe body,基本格式如下：

	probe event {body}

probe event也可以有多个，以逗号隔开，如果有多个event，那么probe body
执行的条件是任意一个探针被触发，可以说是“或”的关系。

<!-- more -->

## 命令行参数

SystemTap允许用户使用简单的命令行参数。使用$表示参数为整数，@表示参数为字符串，如：
```
	probe kernel.function(@1) { }
	probe kernel.function(@1).return {}
```
## event

SystemTap支持很多种类的probe event，具体可见man stapprobes.

systemptap的event主要分为两类，同步与异步, 在设置event时还可以使用通配符“*”
表示所有符合条件的event。

### 同步event

同步event发生在进程执行某一条确定的命令时，比如：

* 系统调用
```
	syscall.system_call
	syscall.system_call.return
	syscall.read
	syscall.read.return
```
* vfs
```
	vfs.file_operation
	vfs.file_operation.return
	vfs.read
	vfs.read.return
```
* 内核函数
```
	kernel.function("function")
	kernel.function("function").return
	kernel.function("sys_open")
	kernel.function("sys_open").return
```
* 文件位置
```
	kernel.function("*@net/socket.c")
	kernel.function("*@net/socket.c").return
```
* tracepoint
```
	kernel.trace("tracepoint")
	kernel.trace("kfree_skb")
```
* 内核模块
```
	module("module").function("function")
	module("module").function("function").return
	probe module("ext3").function("*")
	probe module("ext3").function("*").return
```
### 异步event

异步event不是执行到指定的指令或代码位置，这一类包括计时器，定时器等

* begin
* end
* timer events, 如：

	probe timer.s(4) {statement}

表示每4秒执行一次statement.
也可以有如下格式：

```
	timer.ms(milliseconds)
	timer.us(microseconds)
	timer.ns(nanoseconds)
	timer.hz(hertz)
	timer.jiffies(jiffies)
```

## probe body

### 自定义函数

systemptap也支持函数，定义格式如下：

	function function_name(arguments) {statements}

调用格式如下：

	probe event {function_name(arguments)}

### 内建

* name  系统调用的名称，这个变量只能在probe系统调用时使用
* printf(),可以用来打印信息，与C语言中的printf十分相似，例：
	
	printf("Hello %s\n", arguments)

* tid() The ID of the current thread.
* uid() The ID of the current user.
* cpu() The current CPU number.
* gettimeofday_s() The number of seconds since UNIX epoch (January 1, 1970).
* ctime() Convert number of seconds since UNIX epoch to date.
* pp() A string describing the probe point currently being handled.
* thread_indent() 输出当前probe所处的可执行程序名称、线程id、函数执行的相对时间和执行的次数等
* target() 在stap 脚本通过-x 或-c 指定要检测的任务时使用，如：
	probe syscall.* {
		if (pid() == target())
			printf("%s/n", name)
	}

通过man stapfuncs获取更多函数

### 变量

SystemTap的变量可以全局随处定义，使用。其类型会随着赋值的进行自动识别。它们默认
是局部变量，如果想全区使用，需要关键字global。如：

	global count_ms

### target变量

probe event 会映射到代码的一个确定位置，允许target变量获取该处代码可见变量的值，
可以通过-L选项列出这些变量,如：

	stap -L 'kernel.function("vfs_read")'

如果已经安装debuginfo，会有类似如下的输出：
	
	kernel.function("vfs_read@fs/read_write.c:277") $file:struct file* $buf:char* $count:size_t $pos:loff_t*

每一个target变量都以$开头，”:“后面的表示类型。如果一个target变量不是本地变量，而是
一个全局，外部变量，或是本地定义的静态变量，可能以如下方式表示：

	@var("varname@src/file.c")

#### 检查target变量

随着代码的改变，target 变量也可能会变化。@defined测试是否一个target变量可访问,如：
```
	probe vm.pagefault = kernel.function("__handle_mm_fault@mm/memory.c") ?,
	                     kernel.function("handle_mm_fault@mm/memory.c") ?
		{
		        name = "pagefault"
		        write_access = (@defined($flags) ? $flags & FAULT_FLAG_WRITE : $write_access)
			address =  $address
		}
```
#### ->

SystemTap变量可以使用->获取结构的成员，如：
```
	stap -e 'probe kernel.function("vfs_read") {
		printf ("current files_stat max_files: %d\n",
				@var("files_stat@fs/file_table.c")->max_files);
		exit();
	}'
```

#### 获取内核态数据

对于内核中指向基本数据类型的指针，有一系列的辅助函数可一获取相关值。

* kernel_char(address) Obtain the character at address from kernel memory.
* kernel_short(address) Obtain the short at address from kernel memory.
* kernel_int(address) Obtain the int at address from kernel memory.
* kernel_long(address) Obtain the long at address from kernel memory
* kernel_string(address) Obtain the string at address from kernel memory.
* kernel_string_n(address, n) Obtain the string at address from the kernel memory and limits the string to n bytes.


#### 特殊变量 

$$vars 扩展为 sprintf("parm1=%x ... parmN=%x var1=%x ... varN=%x", parm1, ..., parmN, var1, ..., varN)
参数为probe监视点上的每个变量

$$locals 扩展为本地变量集

$$parms 扩展为函数参数集,如：

	stap -e 'probe kernel.function("vfs_read") {printf("%s\n", $$parms); exit(); }'

vfs_read有4个参数vfs_read: file, buf, count, pos,所以会有类似输出：

	file=0xffff8800b40d4c80 buf=0x7fff634403e0 count=0x2004 pos=0xffff8800af96df48

看到指针没有特别意义，如果想看到具体的值，可以使用“$”作为后缀，如：

	stap -e 'probe kernel.function("vfs_read") {printf("%s\n", $$parms$); exit(); }'

这样就会打印指针域具体的值，会有如下类似输出：

	file={.f_u={...}, .f_path={...}, .f_op=0xffffffffa06e1d80, .f_lock={...}, .f_count={...}, .f_flags=34818, .f_mode=31, .f_pos=0, .f_owner={...}, .f_cred=0xffff88013148fc80, .f_ra={...}, .f_version=0, .f_security=0xffff8800b8dce560, .private_data=0x0, .f_ep_links={...}, .f_mapping=0xffff880037f8fdf8} buf="" count=8196 pos=-131938753921208

使用”$“做为后缀不会扩展内嵌的结构，如果想看到所有的内嵌结构展开，可以使用“$$”后缀，如：

	stap -e 'probe kernel.function("vfs_read") {printf("%s\n", $$parms$$); exit(); }'

会有如下类似输出：

	file={.f_u={.fu_list={.next=0xffff8801336ca0e8, .prev=0xffff88012ded0840}, .fu_rcuhead={.next=0xffff8801336ca0e8, .func=0xffff88012ded0840}}, .f_path={.mnt=0xffff880132fc97c0, .dentry=0xffff88001a889cc0}, .f_op=0xffffffffa06f64c0, .f_lock={.raw_lock={.slock=196611}}, .f_count={.counter=2}, .f_flags=34818, .f_mode=31, .f_pos=0, .f_owner={.lock={.raw_lock={.lock=16777216}}, .pid=0x0, .pid_type=0, .uid=0, .euid=0, .signum=0}, .f_cred=0xffff880130129a80, .f_ra={.start=0, .size=0, .async_size=0, .ra_pages=32, .}}

$$return 扩展为 sprintf("return=%x", $return), 只可以用于return probe.

### 类型转换

大部分情况下，SystemTap可以自动识别变量类型，但是有时会有void类型的变量,
systemtap提供了@cast操作累识别正确类型.@cast的第一个参数是结构的指针，
第二个是cast要转换的目标类型。第三个指定类型定义的文件，如：
```
	function task_state:long (task:long) {
		return @cast(task, "task_struct", "kernel<linux/sched.h>")->state
	}
```

该函数返回long参数指向的task_struct结构的成员state的值.

### 条件语句

可用的条件语句if,while,for, ==, >=, <=, !=等,其语法与C相似，只是语句末尾不需加分号。
