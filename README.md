## 一种基于 http2 ping 的低假阳性的代理检测方法, 使用 golang 实现

### 原理
http2 支持 [PING frame](https://datatracker.ietf.org/doc/html/rfc7540#section-6.7), 一般的浏览器+传输层代理情况下, http2 ping 由浏览器/http2服务器处理, 而 TCP 连接由代理与服务器建立, 此二者延迟视情况可能有差异.  
使用代理的典型情况如下

![image](https://github.com/user-attachments/assets/b5b727dc-8b85-46f2-b2fa-a3c87c177e02)



如抓包高亮所示, 服务器发起 ping 后, +22ms 收到 ack, 再 +34 ms 收到 pong. 这二者时间差便是**黄金指标**, 仅当 http 客户端没有使用代理, 或者代理延迟相当低时, 这两者时间差才趋近于0.  

需要注意, 只有最低延迟才体现连接性质, 比如一个连接 rtt 抖动, 但只要最低 rtt 低至 1ms, 则客户端与服务器一定距离不远, 因此不论 ack rtt 还是 ping rtt, 一轮测试只需要取最低值

### 实现

分成两部分, http2 服务端发起 ping, 和 ack rtt 指标获取. 本例子给出的是为 golang 增加 http2 server ping + 修改 linux 代码以获取最新 rtt.  
除此之外也有其他方式, 例如抓包分析获取 ack rtt, 使用 ebpf 也有可能? 或者不使用 http2, 而是使用 302 跳转并等待新的 syn.

golang 没有 http2 server ping 的接口可用 需要 patch x/net, 参见 [patch](https://github.com/wen-long/go-net/commit/56c8e7d8f9b0e4cb195f97e1d8d35e9fa3bfa632#diff-f930c2ef3e066fdc1a1fdd23284f6d7093374d662fcd5a29a23d5db515ef5191)

Linux 原生的 tcp rtt 是算法持续平滑的,不满足要求. Linux 还有 minrtt, 当网络抖动时只有 ping 测试时段的 rtt 有意义, 也不满足要求. bbr 也有 rtt 指标, 是区间段内取 min, 相比 tcp rtt 更优, 可以作为次佳选择. 为了最好效果, 下面将 tcp rtt 实现进行修改, 以反映最新的 rtt  

基于 [linux-6.10.8](https://cdn.kernel.org/pub/linux/kernel/v6.x/linux-6.10.8.tar.xz) 修改 `linux-6.10.8/net/ipv4/tcp_input.c` 文件

```
648,675c648
< 	u32 new_sample = tp->rcv_rtt_est.rtt_us;
< 	long m = sample;
<
< 	if (new_sample != 0) {
< 		/* If we sample in larger samples in the non-timestamp
< 		 * case, we could grossly overestimate the RTT especially
< 		 * with chatty applications or bulk transfer apps which
< 		 * are stalled on filesystem I/O.
< 		 *
< 		 * Also, since we are only going for a minimum in the
< 		 * non-timestamp case, we do not smooth things out
< 		 * else with timestamps disabled convergence takes too
< 		 * long.
< 		 */
< 		if (!win_dep) {
< 			m -= (new_sample >> 3);
< 			new_sample += m;
< 		} else {
< 			m <<= 3;
< 			if (m < new_sample)
< 				new_sample = m;
< 		}
< 	} else {
< 		/* No previous measure. */
< 		new_sample = m << 3;
< 	}
<
< 	tp->rcv_rtt_est.rtt_us = new_sample;
---
> 	tp->rcv_rtt_est.rtt_us = sample;
716c689
< 	struct tcp_sock *tp = tcp_sk(sk);
---
>         struct tcp_sock *tp = tcp_sk(sk);
718,720c691,693
< 	if (tp->rx_opt.rcv_tsecr == tp->rcv_rtt_last_tsecr)
< 		return;
< 	tp->rcv_rtt_last_tsecr = tp->rx_opt.rcv_tsecr;
---
>         if (tp->rx_opt.rcv_tsecr == tp->rcv_rtt_last_tsecr)
>                 return;
>         tp->rcv_rtt_last_tsecr = tp->rx_opt.rcv_tsecr;
722,724c695
< 	if (TCP_SKB_CB(skb)->end_seq -
< 	    TCP_SKB_CB(skb)->seq >= inet_csk(sk)->icsk_ack.rcv_mss) {
< 		s32 delta = tcp_rtt_tsopt_us(tp);
---
>         s32 delta = tcp_rtt_tsopt_us(tp);
726,728c697,698
< 		if (delta >= 0)
< 			tcp_rcv_rtt_update(tp, delta, 0);
< 	}
---
>         if (delta >= 0)
>                 tcp_rcv_rtt_update(tp, delta, 0);
```


编译内核可使用 defconfig, linux-6.10.8 编译后总共空间占用不到 3GB.
注意需开启 CONFIG_UNIX_DIAG CONFIG_PACKET_DIAG

### 效果

服务端日志
```
*.*.222.164:25308[TW AS13335 Cloudflare], ping rtt 56.76ms, tcp rtt 21.42ms, rttVar 8.03ms
*.*.222.164:25308[TW AS13335 Cloudflare], ping rtt 57.03ms, tcp rtt 21.42ms, rttVar 8.03ms
*.*.222.164:25308[TW AS13335 Cloudflare], ping rtt 57.30ms, tcp rtt 21.42ms, rttVar 8.03ms
*.*.222.164:25308[TW AS13335 Cloudflare], ping rtt 57.02ms, tcp rtt 21.42ms, rttVar 9.55ms
*.*.222.164:25308[TW AS13335 Cloudflare], ping rtt 57.00ms, tcp rtt 21.42ms, rttVar 12.81ms
*.*.222.164:25308[TW AS13335 Cloudflare], ping rtt 57.80ms, tcp rtt 21.42ms, rttVar 12.81ms
*.*.222.164:25308[TW AS13335 Cloudflare], minHttp2Rtt 56.76ms, minTcpRtt 21.42ms
*.*.196.244:14016, ping rtt 334.98ms, tcp rtt 341.20ms, rttVar 97.38ms
*.*.196.244:14016, ping rtt 348.81ms, tcp rtt 339.02ms, rttVar 97.38ms
*.*.196.244:14016, minHttp2Rtt 334.98ms, minTcpRtt 339.02ms
```
浏览器页面
```
经检测: rtt 21.90ms, proxy detected! proxy latency: 35.49ms
经检测: rtt 339.02ms, no proxy detected
```

以上结果均符合实际情况

### 约束

1. 服务端需直接响应请求, 不得有 CDN/nginx 代理
2. 网络抖动越大越难得出结论, 宜部署在与拟检测对象通联状况良好的机器上

### 代理如何避免被检测
可以参考 delayed ACK 的思想, 延迟 ack 的发送, POC:  
修改 linux-6.10.8/net/ipv4/tcp_output.c, 直接干掉 `__tcp_send_ack`
```
4236a4237
>         return;
```

### 实际应用
实际应用一般采用多种指标/方式共同判断, 如结合 IP 信息库, mtu 等

### CREDIT
https://github.com/YCCDSZXH/proxy-checker-rs
