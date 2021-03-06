---
layout: post
title: "火星包: rp_filter and martian packet"
date: 2013-03-18 22:53
comments: true
categories: [linux kernel]
description: "how rp_filter working with linux route and related to martian packet" 
keywords: linux, martian, route 
---
经常在`/var/log/messages`里发现这种消息，它是对流入的包进行路由检查失败后，发出的警告。
```bash
martian source 192.168.1.1 from 10.0.0.1, on dev eth1
ll header: 52:54:00:98:99:d0:52:54:00:de:d8:10:08:00 
```

代码出处在此
```c
kernel_source/net/ipv4/route.c 
static void ip_handle_martian_source(struct net_device *dev,
				     struct in_device *in_dev,
				     struct sk_buff *skb,
				     __be32 daddr,
				     __be32 saddr)
{
	RT_CACHE_STAT_INC(in_martian_src);
#ifdef CONFIG_IP_ROUTE_VERBOSE
	if (IN_DEV_LOG_MARTIANS(in_dev) && net_ratelimit()) {
		/*
		 *	RFC1812 recommendation, if source is martian,
		 *	the only hint is MAC header.
		 */
		printk(KERN_WARNING "martian source %pI4 from %pI4, on dev %s\n",
			&daddr, &saddr, dev->name);
		if (dev->hard_header_len && skb_mac_header_was_set(skb)) {
			int i;
			const unsigned char *p = skb_mac_header(skb);
			printk(KERN_WARNING "ll header: ");
			for (i = 0; i < dev->hard_header_len; i++, p++) {
				printk("%02x", *p);
				if (i < (dev->hard_header_len - 1))
					printk(":");
			}
			printk("\n");
		}
	}
#endif
}
```
<!-- more -->
其中，10.0.0.1表示src ip,192.168.0.1表示dst ip,eth1表示实际收包的设备，来看看为什么会有错误。

```c
kernel_source/net/ipv4/fib_frontend.c
int fib_validate_source(){
    // 代码有选择性省略
    // 反转src和dst
	struct flowi fl = { .nl_u = { .ip4_u =
				      { .daddr = src,
					.saddr = dst,
					.tos = tos } },
			    .mark = mark,
			    .iif = oif };


	in_dev = __in_dev_get_rcu(dev);
    // 拿到收包的设备，顺便取出该设备上rp_filter的flag
	if (in_dev) {
		no_addr = in_dev->ifa_list == NULL;
		rpf = IN_DEV_RPFILTER(in_dev);
		if (mark && !IN_DEV_SRC_VMARK(in_dev))
			fl.mark = 0;
	}
	rcu_read_unlock();

	if (in_dev == NULL)
		goto e_inval;
	net = dev_net(dev);
    // 以src为dst，查fib
	if (fib_lookup(net, &fl, &res))
		goto last_resort;
	if (res.type != RTN_UNICAST)
		goto e_inval_res;
	*spec_dst = FIB_RES_PREFSRC(res);
	fib_combine_itag(itag, &res);

#ifdef CONFIG_IP_ROUTE_MULTIPATH
    // 以src作为dst，查到的发送dev和当前收包的dev相同，一切ok
	if (FIB_RES_DEV(res) == dev || res.fi->fib_nhs > 1)
#else
	if (FIB_RES_DEV(res) == dev)
#endif
	{
		ret = FIB_RES_NH(res).nh_scope >= RT_SCOPE_HOST;
		fib_res_put(&res);
		return ret;
	}
	fib_res_put(&res);
	if (no_addr)
		goto last_resort;
    // 如果dev不相同，并且rp_filter置为on，则检查失败
	if (rpf == 1)
		goto e_inval;

}
```
给一些的配置
```bash
sysctl.conf
// 0 means off,1 means on
net.ipv4.conf.all.rp_filter = 0
net.ipv4.conf.default.rp_filter = 0
net.ipv4.conf.eth0.rp_filter = 0
net.ipv4.conf.lo.rp_filter = 0
net.ipv4.conf.vboxnet0.rp_filter = 0
net.ipv4.conf.wlan0.rp_filter = 0
```
`net.ipv4.conf.all.rp_filter`是总开关，一开全开。

```
// 0 means off
net.ipv4.conf.all.log_martians = 0
net.ipv4.conf.default.log_martians = 0
net.ipv4.conf.eth0.log_martians = 0
net.ipv4.conf.lo.log_martians = 0
net.ipv4.conf.vboxnet0.log_martians = 0
net.ipv4.conf.wlan0.log_martians = 0
```
是否记录martian的开关，对应于代码里的`IN_DEV_LOG_MARTIANS(in_dev)`。

小结一下，这个错误提示是很常见的，以至于常常被忽略，大多数情况它是做了正确的事情，不过当发现有意外的丢包，可以想想是否是遭遇了火星包，排查方法是：先做tcpdump，发现有traffic(tcpdump不受火星包的影响)，但kernel hook或者app收不到包，应怀疑是martian source导致丢包，用dmesg看下是否有相关提示。
```
martian source 192.168.1.1 from 10.0.0.1, on dev eth1
ll header: 52:54:00:98:99:d0:52:54:00:de:d8:10:08:00 
```
做一下翻译:eth1上收到了src=10.0.0.1,dst=192.168.1.1的包，但是按照本机的路由设置对10.0.0.1进行路由计算，得出的out dev不是eth1。
一般遇到这种还是保持rp_filter=1吧，毕竟这个开关能让系统免受很多火星来客的干扰，研究下路由配置应该能解决问题；如果确实很复杂的使用场景，比如这台server有好多个网口，需要在不同网口之间转发，放开rp_filter的限制也无妨。
