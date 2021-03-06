## Background:
When performing stress test on LB, you may often encounter "connect" failures caused by too much client timewait (all ports are occupied in short time). Here are the reasons and solutions.

## Introduction to Linux Parameters:
**tcp_timestamps:** Whether the tcp timestamps option is enabled. timestamps is negotiated during three-way handshake of TCP. If either party does not support it, the timestamps option will not be adopted by the connection.
**tcp_tw_recycle:** Whether reclaiming is enabled for tcp in time_wait status
**tcp_tw_resuse:** If enabled, you can directly reclaim connections which keeps being in time_wait status for more than 1s.

## Reasons:
Too much clients in timewait status. The client actively breaks connections, and each connection breaking changes the status of the connection to timewait. And the reclaiming timeout is 60s by default. Generally, in this case, the user enables parameters `tcp_tw_recycle` and `tcp_tw_resuse` for reclaiming of connections in timewait status.
However, because the `tcp_timestamps` option is disabled in the CLB, neither `tcp_tw_recycle` nor `tcp_tw_resuse` that is enabled by the client can take effect, and connections in timewait status cannot be reclaimed quickly. Next we explain the meaning of several linux parameters and reasons why LB cannot enable `tcp_timestamps`.

1. tcp_tw_recycle and tcp_tw_resuse only take effect when tcp_timestamps is enabled.

2. tcp_timestamps and tcp_tw_recycle cannot be enabled simultaneously, because the public network client may fail to access the server through NAT gateway. The reasons are as follows:

If both tcp_tw_recycle and tcp_timestamps are enabled, the timestamp in the socket connect request of CVM with the same source IP must be incremental within 60s. Taking 2.6.32 kernel as an example, the details are as follows:

![](https://mc.qcloudimg.com/static/img/2199611fec3b323a7b8fd3bb38459913/Linux1.png)

> tmp_opt.saw_tstamp: The socket supports tcp_timestamp
sysctl_tw_recycle: tcp_tw_recycle option is enabled in the local system
TCP_PAWS_MSL: 60s; it indicates that the last TCP communication of the source IP occurred within 60s.
TCP_PAWS_WINDOW: 1; it indicates that the timestamp of the last TCP communication of the source IP is greater than that of this TCP communication.

3. LB (Layer-7) disables tcp_timestamps because public network clients may fail to access to the server through NAT gateway, as shown in the example below:

a)	A quintet is still in the status of time_wait. For allocation policy for port by NAT gateway, the same quintet is re-used within 2MSL, with syn packet sent.

b)	When tcp_timestamps is enabled and the following two conditions are met, the syn packet will be dropped (because the timestamp option is enabled, and it is considered as old packet).
i.	Time stamp of last time > Time stamp of this time
ii.	Packets are received within 24 days (timestamp field is 32-bit and timestamp is updated once per 1 ms by default in Linux. Timestamp echo will occur after 24 days).
Note: This problem is more obvious on the mobile, because all the clients share a limited public IP under the ISP's NAT gateway. The quintet may also be reused within 2MSL. The timestamps sent by different clients may not be incremental.
Taking 2.6.32 kernel as an example, the details are as follows:

![](https://mc.qcloudimg.com/static/img/6228a7dc25c670d4d2fbddc9ea400779/Linux2.png)

> rx_opt->ts_recent: Time stamp of last time
rx_opt->rcv_tsval: Time stamp received this time
get_seconds(): current time
rx_opt->ts_recent_stamp: Time when the last packet was received

## Solutions:
Solutions to the problem of excessive clients in Timewait status:

**1.	When HTTP uses a short connection (Connection: close), LB disables the connection actively, and the client will not generate timewait**.

**2.	If it is required to use a persistent connection, enable SO_LINGER option of the socket and use rst to disable the connection to avoid the timewait status and achieve fast port reclaiming**.

