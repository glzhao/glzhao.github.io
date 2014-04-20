---
layout: post
title: "introducing cephx"
date: 2014-04-21 00:08:56 +0800
comments: true
categories: ceph
---

ceph是一个分布式的存储系统，包括部分的monitors，MDSs,和成千上万的存储节点（OSDs）。ceph的客户端需要不断的和monitor，OSD，甚至MDS连接，因而需要一种授权机制来保证系统安全。ceph中使用的授权机制称为cephx。但ceph授权也不是必须的，如果可以保证网络环境的安全，就可以关闭授权，而且cephx只是针对ceph内部交互的安全，如果需要更加专用或细粒度的授权控制，就需要其他的方式。虽然很小，但激活cephx会对耗费系统一定的资源。cephx的key是以纯文本方式保存在系统中，所以需要注意安全。

<!-- more -->

### 授权过程
client获得monitor的授权过程如图：
![cephx1][1]

除了monitor，client还需要获取MDS（只有cephfs使用）与OSD的授权，过程如下：
![cephx2][2]

### 授权格式
cephx使用caps来描述权限控制，需要注意的是cephfs使用一种不同格式的caps，但只对cephfs有效，不会影响RBD与radosGW。caps可控制的权限如下：

#### allow
授予用户访问权限，如：

    # ceph-authtool -n client.foo --cap mds 'allow'
 
#### r
授予用户读权限，如： 

    # ceph-authtool -n client.foo --cap mon 'allow r'

#### w
授予用户写权限，如：

    # ceph-authtool -n client.foo --cap osd 'allow w'

#### x
授予用户调用class模式的权限，如：

    # ceph-authtool -n client.foo --cap osd 'allow x'

#### class-read
授予用户调用class读的权限，如：

    # ceph-authtool -n client.foo --cap osd 'allow class-read'

#### class-write
授予用户调用class写的权限，如：

    # ceph-authtool -n client.foo --cap osd 'allow class-write'

#### 通配符（*）
授予用户所有权限，包括读写，执行，以及控制权限等，如：

    # ceph-authtool -n client.foo --cap osd 'allow *'

#### 指定pool
除了为指定用户授权之外，还可以为caps指定pool，如：

    # ceph-authtool -n client.foo --cap osd 'allow rwx pool=customer-pool'

### 配置选项
#### 激活授权

    auth cluster required = cephx
    auth service required = cephx
    auth client required = cephx

#### 关闭授权
    
    auth cluster required = none
    auth service required = none
    auth client required = none

授权相关配置选项可见[授权配置文档][3]

  [1]: http://ceph.com/docs/master/_images/ditaa-56e3a72e085f9070289331d64453b84ab1e9510b.png
  [2]: http://ceph.com/docs/master/_images/ditaa-f97566f2e17ba6de07951872d259d25ae061027f.png
  [3]: http://ceph.com/docs/master/rados/configuration/auth-config-ref/
