---
layout: post
title: "systemtap (3) - array"
date: 2014-04-24 22:52:38 +0800
comments: true
categories: tools kernel
---
## 介绍 

systemtap很多时候被用来在线收集信息。因此，数据的组织记录就显得尤其重要，数组
作为一种记录载体很适合这种应用场景。systemtap中的数组相比C，其限制放宽了很多，
功能也更强。比如使用PID，CMD等作为索引，对数组元素的排序等。

<!-- more -->

## 基本使用

systemptap支持数组操作，一般来说，主要用于收集数据后的储存。数组不管是否在多个probe中使用，
都必须被声明为global。其操作方式与awk类似，如：

	array[index]

## 多索引

systemptap中的数组可以支持多达9项索引域,各域间以逗号隔开如：

	device[pid(),execname(),uid(),ppid(),"W"] = devname

使用方式如下：
```
	global reads
	probe vfs.read {
		reads[execname(),pid()] <<< 1
	}
	probe timer.s(3) {
		foreach([var1,var2] in reads)
			printf("%s (%d) : %d \n", var1, var2, @count(reads[var1,var2]))
	}
```

## 操作符号

数组支持 =,  ++ 与 += 操作,其默认的初始值为0，如：

```
	array[tid()] = gettimeofday_s() 
	array[index] ++
	array[index] += $count
```

## foreach

有时候虽然记录了若干的数组元素，但无法获取具体的索引值。foreach操作可以遍历数组所有索引，如：

```
	global reads
	probe vfs.read { 
		  reads[execname()] ++
	}
	probe timer.s(3) {
		foreach (count in reads)
			printf("%s : %d \n", count, reads[count])
       }
```

## limit
		
以上命令将输出reads数组中所有元素，不过是乱序输出，如果想按照一定的顺序，可以使用“-”表示递减，
“+”表示递增。而且，可以通过limit来限制输出元素个数。如下语句将输出最大的10个元素：

```
	probe timer.s(3) {
		foreach (count in reads- limit 10)
			printf("%s : %d \n", count, reads[count])
	}
```

## delete

有时候需要清理不用，或重新使用数组空间，此时可以使用delete来删除数组元素，或整个数组。如：

```
	global reads
	probe vfs.read { 
		reads[execname()] ++
	}
	probe timer.s(3) {
		foreach (count in reads)
			printf("%s : %d \n", count, reads[count])
		delete reads	
	}
```

## in

有时候需要测试某个元素是否在数组中，可以使用in，如：

```
	global reads
	probe vfs.read {
		reads[execname()] ++
	}
	probe timer.s(3) {
		printf("=======\n")
		foreach (count in reads+) 
			printf("%s : %d \n", count, reads[count])
				if(["stapio"] in reads) {
					printf("stapio read detected, exiting\n")
					exit()
				}
	}
```

## 数组统计

为获取数组相关信息，可以使用如下格式：

	@extractor(variable/array index expression)
	
extractor可以是以下任意元素 

### count

Returns the number of all values stored into the variable/array index expression.

	@count(writes[execname()])
	
return how many values are stored in each unique key in array writes.

### sum

Returns the sum of all values stored into the variable/array index expression.

	@sum(writes[execname()])
	
return the total of all values stored in each unique key in array writes.

### min

Returns the smallest among all the values stored in the variable/array index expression.

### max

Returns the largest among all the values stored in the variable/array index expression.

### avg

Returns the average of all values stored in the variable/array index expression.

