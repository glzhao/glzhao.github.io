---
layout: post
title: "tcp半连接之谜"
date: 2014-08-05 12:19:36 +0800
comments: true
categories: kernel
---

## 问题

公司线上遇到了一个问题，TCP建连失败。具体的情况是，client端不断的发送SYN包到server端，通过tcpdump抓包，发现server端也收到了SYN，但就是没有响应。

## 分析

tcp建连的过程不会涉及到用户态的应用层，所以tcpdump可以抓到SYN包，而没有抓到返回的SYN+ACK包说明问题出现在内核网络部分的处理上，必须深入内核代码查看。本文基于Redhat企业版，对应的内核版本为2.6.18-308.24.1.el5。


先来看下TCP协议对应IPV4的结构：

```C
  1310 static struct net_protocol tcp_protocol = {
     1         .handler =      tcp_v4_rcv,
     2         .err_handler =  tcp_v4_err,
     3         .gso_send_check = tcp_v4_gso_send_check,
     4         .gso_segment =  tcp_tso_segment,
     5         .no_policy =    1,
     6 };
```

tcp_v4_crv()为包处理接口：

```C
                ...

  1122         if (!sock_owned_by_user(sk)) {
     1 #ifdef CONFIG_NET_DMA
     2                 struct tcp_sock *tp = tcp_sk(sk);
     3                 if (!tp->ucopy.dma_chan && tp->ucopy.pinned_list)
     4                         tp->ucopy.dma_chan = dma_find_channel(DMA_MEMCPY);
     5                 if (tp->ucopy.dma_chan)
     6                         ret = tcp_v4_do_rcv(sk, skb);
     7                 else
     8 #endif
     9                 {
    10                         if (!tcp_prequeue(sk, skb))
    11                         ret = tcp_v4_do_rcv(sk, skb);
    12                 }
    13         } else if (sk_add_backlog(sk, skb)) {
    14                 bh_unlock_sock(sk);
    15                 goto discard_and_relse;
    16         }

                ...
```

该函数中的tcp_v4_do_rcv(sk, skb)就用来处理接收到的包，再看tcp_v4_do_rcv()对应的定义：

```C
            ...

  1035         TCP_CHECK_TIMER(sk);
     1         if (tcp_rcv_state_process(sk, skb, skb->h.th, skb->len))
     2                 goto reset;
     3         TCP_CHECK_TIMER(sk);
     4         return 0;
            ...
```

tcp_rcv_state_process():

```C
	...

4390         case TCP_LISTEN:
   1                 if(th->ack)
   2                         return;
   3
   4                 if(th->rst)
   5                         goto discard;
   6
   7                 if(th->syn) {
   8                         if (icsk->icsk_af_ops->conn_request(sk, skb) < 0)
   9                                 return 1;
		
	...
```

通过ipv4_specific结构可知icsk->icsk_af_ops->conn_request()对应的callback函数为
tcp_v4_conn_request()

```C
struct inet_connection_sock_af_ops ipv4_specific = {
	.queue_xmit	   = ip_queue_xmit,
	.send_check	   = tcp_v4_send_check,
	.rebuild_header	   = inet_sk_rebuild_header,
	.conn_request	   = tcp_v4_conn_request,
	.syn_recv_sock	   = tcp_v4_syn_recv_sock,
	.remember_stamp	   = tcp_v4_remember_stamp,
	.net_header_len	   = sizeof(struct iphdr),
	.setsockopt	   = ip_setsockopt,
	.getsockopt	   = ip_getsockopt,
	.addr2sockaddr	   = inet_csk_addr2sockaddr,
	.sockaddr_len	   = sizeof(struct sockaddr_in),
```

接下来到了我们这次问题的核心函数tcp_v4_conn_request(),看下定义：

```C
int tcp_v4_conn_request(struct sock *sk, struct sk_buff *skb)
{
	struct inet_request_sock *ireq;
	struct tcp_options_received tmp_opt;
	struct request_sock *req;
	__u32 saddr = skb->nh.iph->saddr;
	__u32 daddr = skb->nh.iph->daddr;
	__u32 isn = TCP_SKB_CB(skb)->when;
	struct dst_entry *dst = NULL;
#ifdef CONFIG_SYN_COOKIES
	int want_cookie = 0;
#else
#define want_cookie 0 /* Argh, why doesn't gcc optimize this :( */
#endif

	/* Never answer to SYNs send to broadcast or multicast */
	if (((struct rtable *)skb->dst)->rt_flags &
	    (RTCF_BROADCAST | RTCF_MULTICAST))
		goto drop;

	/* TW buckets are converted to open requests without
	 * limitations, they conserve resources and peer is
	 * evidently real one.
	 */
	if (inet_csk_reqsk_queue_is_full(sk) && !isn) {
#ifdef CONFIG_SYN_COOKIES
		if (sysctl_tcp_syncookies) {
			want_cookie = 1;
		} else
#endif
		goto drop;
	}

	/* Accept backlog is full. If we have already queued enough
	 * of warm entries in syn queue, drop request. It is better than
	 * clogging syn queue with openreqs with exponentially increasing
	 * timeout.
	 */
	if (sk_acceptq_is_full(sk) && inet_csk_reqsk_queue_young(sk) > 1)
		goto drop;

	req = reqsk_alloc(&tcp_request_sock_ops);
	if (!req)
		goto drop;

	tcp_clear_options(&tmp_opt);
	tmp_opt.mss_clamp = 536;
	tmp_opt.user_mss  = tcp_sk(sk)->rx_opt.user_mss;

	tcp_parse_options(skb, &tmp_opt, 0);

	if (want_cookie && !tmp_opt.saw_tstamp)
		tcp_clear_options(&tmp_opt);

	if (tmp_opt.saw_tstamp && !tmp_opt.rcv_tsval) {
		/* Some OSes (unknown ones, but I see them on web server, which
		 * contains information interesting only for windows'
		 * users) do not send their stamp in SYN. It is easy case.
		 * We simply do not advertise TS support.
		 */
		tmp_opt.saw_tstamp = 0;
		tmp_opt.tstamp_ok  = 0;
	}
	tmp_opt.tstamp_ok = tmp_opt.saw_tstamp;

	tcp_openreq_init(req, &tmp_opt, skb);

	if (security_inet_conn_request(sk, skb, req))
		goto drop_and_free;

	ireq = inet_rsk(req);
	ireq->loc_addr = daddr;
	ireq->rmt_addr = saddr;
	ireq->opt = tcp_v4_save_options(sk, skb);
	if (!want_cookie)
		TCP_ECN_create_request(req, skb->h.th);

	if (want_cookie) {
#ifdef CONFIG_SYN_COOKIES
		syn_flood_warning(skb);
		req->cookie_ts = tmp_opt.tstamp_ok;
#endif
		isn = cookie_v4_init_sequence(sk, skb, &req->mss);
	} else if (!isn) {
		struct inet_peer *peer = NULL;

		/* VJ's idea. We save last timestamp seen
		 * from the destination in peer table, when entering
		 * state TIME-WAIT, and check against it before
		 * accepting new connection request.
		 *
		 * If "isn" is not zero, this request hit alive
		 * timewait bucket, so that all the necessary checks
		 * are made in the function processing timewait state.
		 */
		if (tmp_opt.saw_tstamp &&
		    tcp_death_row.sysctl_tw_recycle &&
		    (dst = inet_csk_route_req(sk, req)) != NULL &&
		    (peer = rt_get_peer((struct rtable *)dst)) != NULL &&
		    peer->v4daddr == saddr) {
			if (xtime.tv_sec < peer->tcp_ts_stamp + TCP_PAWS_MSL &&
			    (s32)(peer->tcp_ts - req->ts_recent) >
							TCP_PAWS_WINDOW) {
				NET_INC_STATS_BH(LINUX_MIB_PAWSPASSIVEREJECTED);
				dst_release(dst);
				goto drop_and_free;
			}
		}
		/* Kill the following clause, if you dislike this way. */
		else if (!sysctl_tcp_syncookies &&
			 (sysctl_max_syn_backlog - inet_csk_reqsk_queue_len(sk) <
			  (sysctl_max_syn_backlog >> 2)) &&
			 (!peer || !peer->tcp_ts_stamp) &&
			 (!dst || !dst_metric(dst, RTAX_RTT))) {
			/* Without syncookies last quarter of
			 * backlog is filled with destinations,
			 * proven to be alive.
			 * It means that we continue to communicate
			 * to destinations, already remembered
			 * to the moment of synflood.
			 */
			LIMIT_NETDEBUG(KERN_DEBUG "TCP: drop open "
				       "request from %u.%u.%u.%u/%u\n",
				       NIPQUAD(saddr),
				       ntohs(skb->h.th->source));
			dst_release(dst);
			goto drop_and_free;
		}

		isn = tcp_v4_init_sequence(sk, skb);
	}
	tcp_rsk(req)->snt_isn = isn;

	if (tcp_v4_send_synack(sk, req, dst))
		goto drop_and_free;

	if (want_cookie) {
	   	reqsk_free(req);
	} else {
		inet_csk_reqsk_queue_hash_add(sk, req, sysctl_tcp_active_timeout);
	}
	return 0;

drop_and_free:
	reqsk_free(req);
drop:
	return 0;
}

```

这上面有很多的drop分支，所以可能有多个丢弃包的可能，由于server的内存充足,所以在排除了其他的几种可能性之后(比如timestamps等),
怀疑问题出在以下两个地方：

```C
	if (inet_csk_reqsk_queue_is_full(sk) && !isn) {
#ifdef CONFIG_SYN_COOKIES
		if (sysctl_tcp_syncookies) {
			want_cookie = 1;
		} else
#endif
		goto drop;
	}

```

和

```C

	if (sk_acceptq_is_full(sk) && inet_csk_reqsk_queue_young(sk) > 1)
		goto drop;

```

由于公司的服务器上开启了synookies ;-(，所以问题极有可能是sk_acceptq_is_full()为真。
我们看下sk_acceptq_is_full()的定义：

```C
static inline int sk_acceptq_is_full(struct sock *sk)
{
	return sk->sk_ack_backlog > sk->sk_max_ack_backlog;
}
```

sk_max_ack_backlog的最大值是如何设置的呢？

sys_listen的实现：

```C
#define SOMAXCONN	128

int sysctl_somaxconn = SOMAXCONN;

asmlinkage long sys_listen(int fd, int backlog)
{
	struct socket *sock;
	int err, fput_needed;
	
	if ((sock = sockfd_lookup_light(fd, &err, &fput_needed)) != NULL) {
		if ((unsigned) backlog > sysctl_somaxconn)
			backlog = sysctl_somaxconn;

		err = security_socket_listen(sock, backlog);
		if (!err)
			err = sock->ops->listen(sock, backlog);

		fput_light(sock->file, fput_needed);
	}
	return err;
}
```

inet_listen()的实现:

```C
int inet_listen(struct socket *sock, int backlog)
{
	struct sock *sk = sock->sk;
	unsigned char old_state;
	int err;

	lock_sock(sk);

	err = -EINVAL;
	if (sock->state != SS_UNCONNECTED || sock->type != SOCK_STREAM)
		goto out;

	old_state = sk->sk_state;
	if (!((1 << old_state) & (TCPF_CLOSE | TCPF_LISTEN)))
		goto out;

	/* Really, if the socket is already in listen state
	 * we can only allow the backlog to be adjusted.
	 */
	if (old_state != TCP_LISTEN) {
		err = inet_csk_listen_start(sk, TCP_SYNQ_HSIZE);
		if (err)
			goto out;
	}
	sk->sk_max_ack_backlog = backlog;
	err = 0;

out:
	release_sock(sk);
	return err;
}
```

从这两个函数可以看出sk_max_ack_backlog的值取两个值的最小值：

* 用户listen()调用的第二个参数backlog
* 系统sk_max_ack_backlog值(对应的proc位置为/proc/sys/net/core/somaxconn, 默认值为128)

syncookies主要是用来应对syc flood攻击的，所以一般情况下不建议开启，这样的话半连接的数量
还受到另外一个条件的影响，inet_csk_reqsk_queue_is_full(sk):

```C
static inline int inet_csk_reqsk_queue_is_full(const struct sock *sk)
{
	return reqsk_queue_is_full(&inet_csk(sk)->icsk_accept_queue);
}

static inline int reqsk_queue_is_full(const struct request_sock_queue *queue)
{
	return queue->listen_opt->qlen >> queue->listen_opt->max_qlen_log;
}
```

我们来看下max_qlen_log是如何赋值的：

```C
int reqsk_queue_alloc(struct request_sock_queue *queue,
		      const int nr_table_entries)
{
	const int lopt_size = sizeof(struct listen_sock) +
			      nr_table_entries * sizeof(struct request_sock *);
	struct listen_sock *lopt = kzalloc(lopt_size, GFP_KERNEL);

	if (lopt == NULL)
		return -ENOMEM;

	for (lopt->max_qlen_log = 6;
	     (1 << lopt->max_qlen_log) < sysctl_max_syn_backlog;
	     lopt->max_qlen_log++);

	get_random_bytes(&lopt->hash_rnd, sizeof(lopt->hash_rnd));
	rwlock_init(&queue->syn_wait_lock);
	queue->rskq_accept_head = NULL;
	lopt->nr_table_entries = nr_table_entries;

	write_lock_bh(&queue->syn_wait_lock);
	queue->listen_opt = lopt;
	write_unlock_bh(&queue->syn_wait_lock);

	return 0;
}
```

max_qlen_log的值受到sysctl_max_syn_backlog(对应的procfs位置为/proc/sys/net/ipv4/tcp_max_syn_backlog)的影响。

## 解决

由此我们可以知道，半连接个数的限制主要在这三个值：

* 用户态调用listen()的第二个参数
* /proc/sys/net/ipv4/tcp_max_syn_backlog
* /proc/sys/net/core/somaxconn

只要调整对应数据就可以大大减少SYN包不响应的问题，不过需要注意以下几点：

1. 如果上层是nginx，其在linux对应的backlog默认值为511，如果网络环境不太好或者并发太大，建议增大
2. 修改对应值时，注意先改两个系统值，再修改对应的应用层参数，否则不生效
3. 调整参数只能减少SYN包不响应出现的频率，不能完全消除。在网络堵塞或者突发性的并发量增大，也可能会出现类似情况
