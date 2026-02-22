.. SPDX-License-Identifier: GPL-2.0

==================================
Linux Network Stack Flow Deep Dive
==================================

:Author: Generated from codebase analysis
:Version: Based on Linux Kernel v6.19

This document provides a comprehensive, code-referenced explanation of how
packets flow through the Linux kernel network stack. Unlike theoretical
overviews, this document references actual function names, source files,
and data structures.

.. contents:: Table of Contents
   :depth: 3

1. High-Level Overview
======================

1.1 Architectural Layers
------------------------

The Linux network stack is organized into distinct layers, each handling
specific responsibilities:

::

    ┌─────────────────────────────────────────────────────────────────┐
    │                     APPLICATION LAYER                          │
    │  (User-space: read/write/send/recv syscalls)                   │
    └─────────────────────────────────────────────────────────────────┘
                                    │
    ┌─────────────────────────────────────────────────────────────────┐
    │                      SOCKET LAYER                               │
    │  Files: net/socket.c, net/ipv4/af_inet.c                       │
    │  Structs: struct socket, struct sock                           │
    └─────────────────────────────────────────────────────────────────┘
                                    │
    ┌─────────────────────────────────────────────────────────────────┐
    │                   TRANSPORT LAYER (L4)                          │
    │  TCP: net/ipv4/tcp*.c (tcp_input.c, tcp_output.c, tcp.c)       │
    │  UDP: net/ipv4/udp.c                                           │
    │  Structs: struct tcp_sock, struct udp_sock                     │
    └─────────────────────────────────────────────────────────────────┘
                                    │
    ┌─────────────────────────────────────────────────────────────────┐
    │                    NETWORK LAYER (L3)                           │
    │  Files: net/ipv4/ip_input.c, ip_output.c, route.c              │
    │  Structs: struct iphdr, struct rtable                          │
    └─────────────────────────────────────────────────────────────────┘
                                    │
    ┌─────────────────────────────────────────────────────────────────┐
    │                   LINK LAYER (L2/Core)                          │
    │  Files: net/core/dev.c, net/core/skbuff.c                      │
    │  Structs: struct sk_buff, struct net_device                    │
    └─────────────────────────────────────────────────────────────────┘
                                    │
    ┌─────────────────────────────────────────────────────────────────┐
    │                      DRIVER LAYER                               │
    │  Directory: drivers/net/*                                       │
    │  Example: drivers/net/loopback.c, drivers/net/virtio_net.c     │
    └─────────────────────────────────────────────────────────────────┘
                                    │
    ┌─────────────────────────────────────────────────────────────────┐
    │                        HARDWARE                                 │
    │  (NIC, Physical Network Interface)                              │
    └─────────────────────────────────────────────────────────────────┘


1.2 Key Components and Files
----------------------------

**Core Network Layer:**

- ``net/core/dev.c``: Main network device handling, packet reception/transmission
- ``net/core/skbuff.c``: Socket buffer (sk_buff) allocation and manipulation
- ``net/core/sock.c``: Generic socket operations
- ``net/core/flow_dissector.c``: Packet flow analysis

**IPv4 Layer:**

- ``net/ipv4/ip_input.c``: IP packet reception (``ip_rcv()``, ``ip_local_deliver()``)
- ``net/ipv4/ip_output.c``: IP packet transmission (``ip_queue_xmit()``, ``ip_local_out()``)
- ``net/ipv4/route.c``: Routing table and decisions
- ``net/ipv4/arp.c``: ARP protocol handling

**TCP Layer:**

- ``net/ipv4/tcp.c``: Main TCP implementation (``tcp_sendmsg()``, ``tcp_recvmsg()``)
- ``net/ipv4/tcp_input.c``: TCP receive processing (``tcp_rcv_state_process()``)
- ``net/ipv4/tcp_output.c``: TCP send processing (``tcp_write_xmit()``)
- ``net/ipv4/tcp_ipv4.c``: IPv4-specific TCP (``tcp_v4_rcv()``)
- ``net/ipv4/tcp_timer.c``: TCP timers and retransmission

**Socket Layer:**

- ``net/socket.c``: System call entry points
- ``net/ipv4/af_inet.c``: AF_INET socket family

**Headers:**

- ``include/linux/skbuff.h``: struct sk_buff definition
- ``include/net/sock.h``: struct sock definition
- ``include/net/tcp.h``: TCP constants and macros
- ``include/net/tcp_states.h``: TCP state machine enum
- ``include/linux/netdevice.h``: struct net_device, NAPI


2. End-to-End RX Flow: TCP Packet from Wire to Application
===========================================================

2.1 Stage 1: Hardware & Driver (Interrupt Context)
---------------------------------------------------

When a packet arrives at the NIC:

1. **Hardware receives frame** and places it in a ring buffer (DMA)
2. **NIC raises an interrupt** to notify the CPU
3. **Driver ISR executes**, typically calling ``napi_schedule()``

**Example from drivers/net/virtio_net.c:**

The driver interrupt handler schedules NAPI polling::

    // In driver interrupt handler:
    if (napi_schedule_prep(napi))
        __napi_schedule(napi);

**Key function: ____napi_schedule()** in ``net/core/dev.c:4915``::

    static inline void ____napi_schedule(struct softnet_data *sd,
                                          struct napi_struct *napi)
    {
        struct task_struct *thread;

        lockdep_assert_irqs_disabled();

        if (test_bit(NAPI_STATE_THREADED, &napi->state)) {
            // Threaded NAPI mode
            wake_up_process(napi->thread);
            return;
        }

        // Add to poll_list and raise softirq
        list_add_tail(&napi->poll_list, &sd->poll_list);
        __raise_softirq_irqoff(NET_RX_SOFTIRQ);
    }


2.2 Stage 2: NAPI Poll (Softirq Context)
----------------------------------------

The NET_RX_SOFTIRQ runs ``net_rx_action()``, which calls the driver's poll
function.

**net_rx_action()** in ``net/core/dev.c``::

    static __latent_entropy void net_rx_action(void)
    {
        // Process NAPI poll list
        for (;;) {
            struct napi_struct *n = list_first_entry(...);
            work = __napi_poll(n, &repoll);
            budget -= work;
        }
    }

**__napi_poll()** in ``net/core/dev.c:7670``::

    static int __napi_poll(struct napi_struct *n, bool *repoll)
    {
        // Call driver's poll function
        work = n->poll(n, weight);

        if (work < weight) {
            // Polling complete, re-enable interrupts
            napi_complete_done(n, work);
        }
        return work;
    }

The driver's poll function retrieves packets and calls ``napi_gro_receive()``
or ``netif_receive_skb()`` for each packet.


2.3 Stage 3: Generic Receive Processing
---------------------------------------

**netif_receive_skb()** in ``net/core/dev.c:6408`` is the main entry point::

    int netif_receive_skb(struct sk_buff *skb)
    {
        trace_netif_receive_skb_entry(skb);
        ret = netif_receive_skb_internal(skb);
        trace_netif_receive_skb_exit(ret);
        return ret;
    }

This leads to **__netif_receive_skb_core()** in ``net/core/dev.c:5926``::

    static int __netif_receive_skb_core(struct sk_buff **pskb, bool pfmemalloc,
                                        struct packet_type **ppt_prev)
    {
        struct sk_buff *skb = *pskb;

        // Reset headers
        skb_reset_network_header(skb);
        skb_reset_mac_len(skb);

        // Process XDP if enabled
        if (static_branch_unlikely(&generic_xdp_needed_key)) {
            ret = do_xdp_generic(skb->dev->xdp_prog, &skb);
        }

        // Handle VLAN
        if (eth_type_vlan(skb->protocol)) {
            skb = skb_vlan_untag(skb);
        }

        // TC/ingress processing
        skb = sch_handle_ingress(skb, ...);

        // Call rx_handler (bridging, bonding)
        rx_handler = rcu_dereference(skb->dev->rx_handler);
        if (rx_handler) {
            switch (rx_handler(&skb)) {
            case RX_HANDLER_CONSUMED:
                goto out;
            }
        }

        // Deliver to protocol handler based on skb->protocol
        type = skb->protocol;
        deliver_ptype_list_skb(skb, &pt_prev, orig_dev, type, ...);
    }

The protocol handler is found in ``ptype_base[hash]`` hash table, registered
by IP layer at boot time.


2.4 Stage 4: IP Layer Processing
--------------------------------

For IPv4, **ip_rcv()** is registered as the handler for ETH_P_IP::

    // Registration in net/ipv4/af_inet.c:1886
    static struct packet_type ip_packet_type __read_mostly = {
        .type = cpu_to_be16(ETH_P_IP),
        .func = ip_rcv,
    };

**ip_rcv()** in ``net/ipv4/ip_input.c:564``::

    int ip_rcv(struct sk_buff *skb, struct net_device *dev,
               struct packet_type *pt, struct net_device *orig_dev)
    {
        struct net *net = dev_net(dev);

        skb = ip_rcv_core(skb, net);  // Validate IP header
        if (!skb)
            return NET_RX_DROP;

        // Pass through netfilter (PRE_ROUTING hook)
        return NF_HOOK(NFPROTO_IPV4, NF_INET_PRE_ROUTING,
                       net, NULL, skb, dev, NULL, ip_rcv_finish);
    }

**ip_rcv_core()** performs validation::

    static struct sk_buff *ip_rcv_core(struct sk_buff *skb, struct net *net)
    {
        const struct iphdr *iph;

        // Ensure we can read IP header
        if (!pskb_may_pull(skb, sizeof(struct iphdr)))
            goto drop;

        iph = ip_hdr(skb);

        // Validate version and header length
        if (iph->ihl < 5 || iph->version != 4)
            goto drop;

        // Validate total length
        if (ntohs(iph->tot_len) < (iph->ihl * 4))
            goto drop;

        // Verify IP checksum
        if (ip_fast_csum((u8 *)iph, iph->ihl))
            goto csum_error;

        return skb;
    }

**ip_local_deliver()** for locally-destined packets::

    int ip_local_deliver(struct sk_buff *skb)
    {
        struct net *net = dev_net(skb->dev);

        // Defragment if necessary
        if (ip_is_fragment(ip_hdr(skb))) {
            skb = ip_defrag(net, skb, IP_DEFRAG_LOCAL_DELIVER);
            if (!skb)
                return 0;
        }

        // Pass through netfilter (LOCAL_IN hook)
        return NF_HOOK(NFPROTO_IPV4, NF_INET_LOCAL_IN,
                       net, NULL, skb, skb->dev, NULL,
                       ip_local_deliver_finish);
    }

**ip_protocol_deliver_rcu()** dispatches to transport protocol::

    void ip_protocol_deliver_rcu(struct net *net, struct sk_buff *skb,
                                  int protocol)
    {
        const struct net_protocol *ipprot;

        // Lookup protocol handler in inet_protos table
        ipprot = rcu_dereference(inet_protos[protocol]);
        if (ipprot) {
            // Direct call with optimization for TCP/UDP
            ret = INDIRECT_CALL_2(ipprot->handler,
                                  tcp_v4_rcv, udp_rcv, skb);
        }
    }


2.5 Stage 5: TCP Layer Processing
---------------------------------

**tcp_v4_rcv()** in ``net/ipv4/tcp_ipv4.c:2143`` is the TCP entry point::

    int tcp_v4_rcv(struct sk_buff *skb)
    {
        struct net *net = dev_net(skb->dev);
        const struct iphdr *iph;
        const struct tcphdr *th;
        struct sock *sk;

        // Count incoming segment
        __TCP_INC_STATS(net, TCP_MIB_INSEGS);

        // Ensure we can read TCP header
        if (!pskb_may_pull(skb, sizeof(struct tcphdr)))
            goto discard_it;

        th = (const struct tcphdr *)skb->data;

        // Validate header length
        if (th->doff < sizeof(struct tcphdr) / 4)
            goto bad_packet;

        // Initialize checksum validation
        if (skb_checksum_init(skb, IPPROTO_TCP, inet_compute_pseudo))
            goto csum_error;

        // Lookup socket by 4-tuple
        sk = __inet_lookup_skb(skb, th->source, th->dest, ...);
        if (!sk)
            goto no_tcp_socket;

        // Handle different socket states
        if (sk->sk_state == TCP_TIME_WAIT)
            goto do_time_wait;

        if (sk->sk_state == TCP_NEW_SYN_RECV) {
            // Handle SYN-ACK for connection being established
            req = inet_reqsk(sk);
            nsk = tcp_check_req(sk, skb, req, ...);
        }

        // Lock socket and process
        bh_lock_sock_nested(sk);
        if (!sock_owned_by_user(sk)) {
            ret = tcp_v4_do_rcv(sk, skb);
        } else {
            // Backlog processing if socket is locked by user
            sk_add_backlog(sk, skb, ...);
        }
        bh_unlock_sock(sk);

        return ret;
    }

**tcp_v4_do_rcv()** routes to appropriate handler::

    int tcp_v4_do_rcv(struct sock *sk, struct sk_buff *skb)
    {
        if (sk->sk_state == TCP_ESTABLISHED) {
            // Fast path for established connections
            tcp_rcv_established(sk, skb);
            return 0;
        }

        // Slow path for other states
        if (tcp_rcv_state_process(sk, skb)) {
            // Error handling
        }
        return 0;
    }

**tcp_rcv_established()** in ``net/ipv4/tcp_input.c`` is the fast path::

    void tcp_rcv_established(struct sock *sk, struct sk_buff *skb)
    {
        struct tcp_sock *tp = tcp_sk(sk);

        // Header prediction - optimize for common case
        if ((tcp_flag_word(th) & TCP_HP_BITS) == tp->pred_flags &&
            TCP_SKB_CB(skb)->seq == tp->rcv_nxt &&
            !after(TCP_SKB_CB(skb)->ack_seq, tp->snd_nxt)) {

            // Fast path: in-order data
            if (len <= tcp_header_len) {
                // Pure ACK
                tcp_ack(sk, skb, 0);
            } else {
                // Data packet
                __skb_queue_tail(&sk->sk_receive_queue, skb);
                tp->rcv_nxt = TCP_SKB_CB(skb)->end_seq;
                sk->sk_data_ready(sk);  // Wake up reader
            }
            return;
        }

        // Slow path for complex cases (OOO, SACK, etc.)
        tcp_data_queue(sk, skb);
    }


2.6 Stage 6: Socket Layer & Application Wake-up
------------------------------------------------

Data is queued to ``sk->sk_receive_queue`` and the application is notified::

    sk->sk_data_ready(sk);  // Calls sock_def_readable()

**sock_def_readable()** in ``net/core/sock.c``::

    void sock_def_readable(struct sock *sk)
    {
        struct socket_wq *wq;

        rcu_read_lock();
        wq = rcu_dereference(sk->sk_wq);
        if (skwq_has_sleeper(wq))
            wake_up_interruptible_sync_poll(&wq->wait, ...);
        rcu_read_unlock();
    }

Application reads via **tcp_recvmsg()** in ``net/ipv4/tcp.c``::

    int tcp_recvmsg(struct sock *sk, struct msghdr *msg, size_t len,
                    int flags, int *addr_len)
    {
        struct tcp_sock *tp = tcp_sk(sk);
        struct sk_buff *skb;

        lock_sock(sk);

        // Wait for data
        while (!(skb = tcp_recv_skb(sk, seq, &offset))) {
            // Sleep until data arrives
            sk_wait_data(sk, &timeo, last);
        }

        // Copy data to user buffer
        err = skb_copy_datagram_msg(skb, offset, msg, used);

        // Update sequence numbers
        tp->copied_seq += used;

        release_sock(sk);
        return copied;
    }


3. TX Path: Application to Hardware
====================================

3.1 System Call Entry
---------------------

Application calls ``send()`` or ``write()`` which enters via::

    // net/socket.c
    SYSCALL_DEFINE6(sendto, ...)
    {
        sock_sendmsg(sock, &msg);
    }

    int sock_sendmsg(struct socket *sock, struct msghdr *msg)
    {
        return sock->ops->sendmsg(sock, msg, msg_data_left(msg));
    }


3.2 TCP Send Path
-----------------

**tcp_sendmsg()** in ``net/ipv4/tcp.c:1459``::

    int tcp_sendmsg(struct sock *sk, struct msghdr *msg, size_t size)
    {
        lock_sock(sk);
        ret = tcp_sendmsg_locked(sk, msg, size);
        release_sock(sk);
        return ret;
    }

**tcp_sendmsg_locked()** in ``net/ipv4/tcp.c:1129``::

    int tcp_sendmsg_locked(struct sock *sk, struct msghdr *msg, size_t size)
    {
        struct tcp_sock *tp = tcp_sk(sk);
        struct sk_buff *skb;

        // Wait for connection if needed
        if (((1 << sk->sk_state) & ~(TCPF_ESTABLISHED | TCPF_CLOSE_WAIT))) {
            err = sk_stream_wait_connect(sk, &timeo);
        }

        // Calculate MSS
        mss_now = tcp_send_mss(sk, &size_goal, flags);

        while (msg_data_left(msg)) {
            // Get or create skb for write queue
            skb = tcp_write_queue_tail(sk);
            if (!skb || tcp_skb_is_last(sk, skb)) {
                skb = tcp_stream_alloc_skb(sk, ...);
                skb_entail(sk, skb);  // Add to sk_write_queue
            }

            // Copy data from user space
            copy = min_t(int, msg_data_left(msg), size_goal - skb->len);
            err = skb_add_data_nocache(sk, skb, &msg->msg_iter, copy);

            // Set sequence numbers
            TCP_SKB_CB(skb)->end_seq = tp->write_seq;
            copied += copy;
        }

        // Try to push data out
        tcp_push(sk, flags, mss_now, ...);

        return copied;
    }

**tcp_push()** triggers transmission::

    void tcp_push(struct sock *sk, int flags, int mss_now, ...)
    {
        struct tcp_sock *tp = tcp_sk(sk);

        // Set PSH flag if requested
        if (flags & MSG_MORE)
            nonagle = TCP_NAGLE_CORK;

        // Call output engine
        __tcp_push_pending_frames(sk, mss_now, nonagle);
    }


3.3 TCP Output Engine
---------------------

**tcp_write_xmit()** in ``net/ipv4/tcp_output.c:2966`` is the main output function::

    static bool tcp_write_xmit(struct sock *sk, unsigned int mss_now,
                                int nonagle, int push_one, gfp_t gfp)
    {
        struct tcp_sock *tp = tcp_sk(sk);
        struct sk_buff *skb;
        unsigned int sent_pkts;

        while ((skb = tcp_send_head(sk))) {
            // Check congestion window
            cwnd_quota = tcp_cwnd_test(tp, skb);
            if (!cwnd_quota)
                break;

            // Check receiver window
            if (!tcp_snd_wnd_test(tp, skb, mss_now))
                break;

            // TSO segmentation if needed
            if (skb->len > mss_now) {
                tcp_fragment(sk, skb, mss_now, ...);
            }

            // Transmit the segment
            err = tcp_transmit_skb(sk, skb, 1, gfp);
            if (err)
                break;

            // Update send state
            tcp_event_new_data_sent(sk, skb);
            sent_pkts++;
        }

        return sent_pkts > 0;
    }

**tcp_transmit_skb()** builds the TCP header::

    static int __tcp_transmit_skb(struct sock *sk, struct sk_buff *skb,
                                   int clone_it, gfp_t gfp_mask, u32 rcv_nxt)
    {
        struct inet_connection_sock *icsk = inet_csk(sk);
        struct tcp_sock *tp = tcp_sk(sk);
        struct tcphdr *th;

        // Clone skb if needed (for retransmission)
        if (clone_it) {
            skb = skb_clone(skb, gfp_mask);
        }

        // Reserve space for TCP header
        skb_push(skb, tcp_header_size);
        skb_reset_transport_header(skb);

        // Build TCP header
        th = (struct tcphdr *)skb->data;
        th->source = inet->inet_sport;
        th->dest   = inet->inet_dport;
        th->seq    = htonl(TCP_SKB_CB(skb)->seq);
        th->ack_seq = htonl(rcv_nxt);
        th->doff   = tcp_header_size >> 2;
        th->window = htons(tcp_select_window(sk));

        // Set flags
        tcp_set_flags(th, TCP_SKB_CB(skb)->tcp_flags);

        // Add TCP options (timestamps, SACK, etc.)
        tcp_options_write(th, tp, &opts, &key);

        // Calculate checksum (or set up for HW offload)
        icsk->icsk_af_ops->send_check(sk, skb);

        // Pass to IP layer
        err = INDIRECT_CALL_INET(icsk->icsk_af_ops->queue_xmit,
                                 inet6_csk_xmit, ip_queue_xmit,
                                 sk, skb, &inet->cork.fl);
        return err;
    }


3.4 IP Layer TX
---------------

**ip_queue_xmit()** in ``net/ipv4/ip_output.c:546``::

    int ip_queue_xmit(struct sock *sk, struct sk_buff *skb, struct flowi *fl)
    {
        return __ip_queue_xmit(sk, skb, fl, inet_sk(sk)->tos);
    }

**__ip_queue_xmit()** in ``net/ipv4/ip_output.c:463``::

    int __ip_queue_xmit(struct sock *sk, struct sk_buff *skb,
                        struct flowi *fl, __u8 tos)
    {
        struct inet_sock *inet = inet_sk(sk);
        struct rtable *rt;
        struct iphdr *iph;

        // Get route (cached in socket or lookup)
        rt = ip_route_output_ports(net, fl4, sk, ...);
        skb_dst_set_noref(skb, &rt->dst);

        // Push IP header
        skb_push(skb, sizeof(struct iphdr));
        skb_reset_network_header(skb);

        // Build IP header
        iph = ip_hdr(skb);
        iph->version = 4;
        iph->ihl     = 5;
        iph->tos     = tos;
        iph->ttl     = ip_select_ttl(inet, &rt->dst);
        iph->protocol = sk->sk_protocol;  // IPPROTO_TCP
        iph->saddr   = fl4->saddr;
        iph->daddr   = fl4->daddr;
        iph->frag_off = htons(IP_DF);
        ip_select_ident(net, skb, NULL);

        // Pass to local output
        return ip_local_out(net, sk, skb);
    }

**ip_local_out()** calculates checksum and sends::

    int ip_local_out(struct net *net, struct sock *sk, struct sk_buff *skb)
    {
        // Calculate IP checksum
        ip_send_check(ip_hdr(skb));

        // Pass through netfilter (LOCAL_OUT hook)
        return NF_HOOK(NFPROTO_IPV4, NF_INET_LOCAL_OUT,
                       net, sk, skb, NULL, skb_dst_dev(skb),
                       dst_output);
    }

**ip_send_check()** in ``net/ipv4/ip_output.c:95``::

    void ip_send_check(struct iphdr *iph)
    {
        iph->check = 0;
        iph->check = ip_fast_csum((unsigned char *)iph, iph->ihl);
    }


3.5 Link Layer TX
-----------------

**dst_output()** eventually calls **dev_queue_xmit()**.

**__dev_queue_xmit()** in ``net/core/dev.c:4743``::

    int __dev_queue_xmit(struct sk_buff *skb, struct net_device *sb_dev)
    {
        struct net_device *dev = skb->dev;
        struct netdev_queue *txq;

        // Get TX queue
        txq = netdev_core_pick_tx(dev, skb, sb_dev);

        // TC egress processing
        skb = sch_handle_egress(skb, &rc, dev);

        // Direct transmit if no qdisc
        if (q->enqueue) {
            rc = __dev_xmit_skb(skb, q, dev, txq);
        } else {
            // No qdisc, direct transmit
            rc = dev_hard_start_xmit(skb, dev, txq, &rc);
        }

        return rc;
    }

**dev_hard_start_xmit()** calls driver's ndo_start_xmit::

    struct sk_buff *dev_hard_start_xmit(struct sk_buff *skb,
                                         struct net_device *dev, ...)
    {
        const struct net_device_ops *ops = dev->netdev_ops;

        // Call driver's transmit function
        rc = ops->ndo_start_xmit(skb, dev);

        return skb;
    }


3.6 Driver TX
-------------

Example from loopback driver in ``drivers/net/loopback.c:70``::

    static netdev_tx_t loopback_xmit(struct sk_buff *skb,
                                      struct net_device *dev)
    {
        skb_tx_timestamp(skb);
        skb_orphan(skb);

        // Loopback directly receives the packet
        skb->protocol = eth_type_trans(skb, dev);

        if (likely(__netif_rx(skb) == NET_RX_SUCCESS))
            dev_lstats_add(dev, len);

        return NETDEV_TX_OK;
    }


4. Memory & Buffers
====================

4.1 The sk_buff Structure
-------------------------

The ``struct sk_buff`` (socket buffer) is the fundamental packet
representation in Linux. Defined in ``include/linux/skbuff.h:885``::

    struct sk_buff {
        /* List management */
        union {
            struct {
                struct sk_buff      *next;
                struct sk_buff      *prev;
                struct net_device   *dev;
            };
            struct rb_node    rbnode;   /* Used in TCP retransmit queue */
            struct list_head  list;
        };

        struct sock     *sk;            /* Owning socket */

        /* Timestamps */
        union {
            ktime_t     tstamp;
            u64         skb_mstamp_ns;  /* For TCP RTT measurement */
        };

        /* Per-layer control data (48 bytes) */
        char            cb[48] __aligned(8);

        /* Routing information */
        unsigned long   _skb_refdst;
        void            (*destructor)(struct sk_buff *skb);

        /* Length information */
        unsigned int    len;            /* Total length of data */
        unsigned int    data_len;       /* Length of paged data */
        __u16           mac_len;        /* Length of MAC header */
        __u16           hdr_len;        /* Writable header length */

        /* Checksum information */
        __u16           csum_start;
        __u16           csum_offset;
        __wsum          csum;

        /* Protocol and state */
        __u8            pkt_type:3;     /* PACKET_HOST, etc. */
        __u8            ip_summed:2;    /* Checksum status */
        __u8            cloned:1;
        __u8            nohdr:1;
        __be16          protocol;       /* ETH_P_IP, etc. */

        /* Header offsets */
        __u16           transport_header;
        __u16           network_header;
        __u16           mac_header;

        /* Buffer pointers */
        sk_buff_data_t  tail;
        sk_buff_data_t  end;
        unsigned char   *head;
        unsigned char   *data;

        unsigned int    truesize;       /* Actual allocated size */
        refcount_t      users;          /* Reference count */
    };


4.2 Buffer Layout
-----------------

::

                                  ---------------
                                 | sk_buff       |
                                  ---------------
     ,---------------------------  + head
    /          ,-----------------  + data
   /          /      ,-----------  + tail
  |          |      |            , + end
  |          |      |           |
  v          v      v           v
   -----------------------------------------------
  | headroom | data |  tailroom | skb_shared_info |
   -----------------------------------------------
                                 + [page frag]
                                 + [page frag]
                                 + [page frag]
                                 + frag_list    --> | sk_buff |


4.3 Buffer Allocation
---------------------

**alloc_skb()** in ``net/core/skbuff.c``::

    struct sk_buff *__alloc_skb(unsigned int size, gfp_t gfp_mask,
                                 int flags, int node)
    {
        struct sk_buff *skb;
        u8 *data;

        // Allocate sk_buff structure from cache
        skb = kmem_cache_alloc_node(skbuff_cache, gfp_mask, node);

        // Allocate data buffer
        data = kmalloc_reserve(&size, gfp_mask, node, ...);

        // Initialize pointers
        skb->head = data;
        skb->data = data;
        skb->tail = data;
        skb->end = data + size;

        return skb;
    }


4.4 Header Manipulation
-----------------------

Push data (add header)::

    unsigned char *skb_push(struct sk_buff *skb, unsigned int len)
    {
        skb->data -= len;
        skb->len  += len;
        return skb->data;
    }

Pull data (remove header)::

    unsigned char *skb_pull(struct sk_buff *skb, unsigned int len)
    {
        skb->len -= len;
        return skb->data += len;
    }

Header pointer access::

    static inline struct iphdr *ip_hdr(const struct sk_buff *skb)
    {
        return (struct iphdr *)skb_network_header(skb);
    }

    static inline struct tcphdr *tcp_hdr(const struct sk_buff *skb)
    {
        return (struct tcphdr *)skb_transport_header(skb);
    }


4.5 Zero-Copy Support
---------------------

The kernel supports zero-copy in several ways:

1. **MSG_ZEROCOPY for TX**: User pages are directly used via scatter-gather
2. **Paged sk_buff fragments**: Data in pages, not linear buffer
3. **splice/sendfile**: Move data without copying through user space

For TCP sendmsg with MSG_ZEROCOPY (``net/ipv4/tcp.c:1153``)::

    if (flags & MSG_ZEROCOPY) {
        uarg = msg_zerocopy_realloc(sk, size, skb_zcopy(skb), ...);
        if (sk->sk_route_caps & NETIF_F_SG)
            zc = MSG_ZEROCOPY;
    }


5. TCP State Machine
====================

5.1 State Definition
--------------------

From ``include/net/tcp_states.h``::

    enum {
        TCP_ESTABLISHED = 1,
        TCP_SYN_SENT,
        TCP_SYN_RECV,
        TCP_FIN_WAIT1,
        TCP_FIN_WAIT2,
        TCP_TIME_WAIT,
        TCP_CLOSE,
        TCP_CLOSE_WAIT,
        TCP_LAST_ACK,
        TCP_LISTEN,
        TCP_CLOSING,
        TCP_NEW_SYN_RECV,   /* Mini socket for SYN+ACK */
        TCP_BOUND_INACTIVE, /* Pseudo-state for inet_diag */
        TCP_MAX_STATES
    };

State flags for matching multiple states::

    enum {
        TCPF_ESTABLISHED = (1 << TCP_ESTABLISHED),
        TCPF_SYN_SENT    = (1 << TCP_SYN_SENT),
        TCPF_SYN_RECV    = (1 << TCP_SYN_RECV),
        // ... etc
    };


5.2 State Machine Implementation
--------------------------------

**tcp_rcv_state_process()** in ``net/ipv4/tcp_input.c:7172`` handles
all states except ESTABLISHED and TIME_WAIT::

    enum skb_drop_reason
    tcp_rcv_state_process(struct sock *sk, struct sk_buff *skb)
    {
        struct tcp_sock *tp = tcp_sk(sk);
        const struct tcphdr *th = tcp_hdr(skb);

        switch (sk->sk_state) {
        case TCP_CLOSE:
            goto discard;

        case TCP_LISTEN:
            if (th->ack)
                return SKB_DROP_REASON_TCP_FLAGS;
            if (th->rst)
                goto discard;
            if (th->syn) {
                // Handle incoming SYN
                icsk->icsk_af_ops->conn_request(sk, skb);
                return 0;
            }
            goto discard;

        case TCP_SYN_SENT:
            // Handle SYN-ACK
            queued = tcp_rcv_synsent_state_process(sk, skb, th);
            return queued;

        case TCP_SYN_RECV:
        case TCP_FIN_WAIT1:
        case TCP_FIN_WAIT2:
        case TCP_CLOSE_WAIT:
        case TCP_CLOSING:
        case TCP_LAST_ACK:
            // Process ACK
            if (!tcp_validate_incoming(sk, skb, th, 0))
                return 0;
            // ... handle each state
        }
    }


5.3 Three-Way Handshake (Passive Open - Server)
-----------------------------------------------

**Step 1: SYN Received (TCP_LISTEN state)**

Server socket is in LISTEN state. When SYN arrives::

    case TCP_LISTEN:
        if (th->syn) {
            icsk->icsk_af_ops->conn_request(sk, skb);
        }

``conn_request`` points to **tcp_v4_conn_request()** which calls
**tcp_conn_request()** in ``net/ipv4/tcp_input.c:7642``::

    int tcp_conn_request(struct request_sock_ops *rsk_ops,
                         const struct tcp_request_sock_ops *af_ops,
                         struct sock *sk, struct sk_buff *skb)
    {
        struct request_sock *req;

        // Allocate request socket (mini socket)
        req = inet_reqsk_alloc(rsk_ops, sk, !want_cookie);

        // Parse TCP options from SYN
        tcp_parse_options(skb, &tmp_opt, ...);

        // Initialize request socket
        tcp_openreq_init(req, &tmp_opt, skb, sk);

        // Generate initial sequence number
        isn = af_ops->init_seq(skb);

        // Send SYN-ACK
        af_ops->send_synack(sk, dst, req, ...);

        // Add to SYN queue
        inet_csk_reqsk_queue_hash_add(sk, req, timeout);

        return 0;
    }

**Step 2: SYN-ACK Sent**

``send_synack`` points to **tcp_v4_send_synack()**::

    static int tcp_v4_send_synack(const struct sock *sk, ...)
    {
        // Build SYN-ACK packet
        skb = tcp_make_synack(sk, dst, req, ...);

        // Transmit
        err = ip_build_and_send_pkt(skb, sk, ...);
    }

**Step 3: ACK Received**

When the final ACK arrives, it's handled in **tcp_v4_rcv()**::

    if (sk->sk_state == TCP_NEW_SYN_RECV) {
        struct request_sock *req = inet_reqsk(sk);

        // Create full socket from request socket
        nsk = tcp_check_req(sk, skb, req, ...);

        // tcp_check_req() calls:
        // child = inet_csk(sk)->icsk_af_ops->syn_recv_sock(sk, skb, req);
        // This creates the new full socket in ESTABLISHED state
    }

**tcp_create_openreq_child()** in ``net/ipv4/tcp_minisocks.c:549``
creates the child socket::

    struct sock *tcp_create_openreq_child(const struct sock *sk,
                                           struct request_sock *req,
                                           struct sk_buff *skb)
    {
        struct sock *newsk = inet_csk_clone_lock(sk, req, ...);
        struct tcp_sock *newtp = tcp_sk(newsk);

        // Initialize child socket
        newtp->snd_nxt = treq->snt_isn + 1;
        newtp->rcv_nxt = treq->rcv_isn + 1;

        // Set state to ESTABLISHED
        tcp_init_transfer(newsk, ...);

        return newsk;
    }


5.4 Three-Way Handshake (Active Open - Client)
----------------------------------------------

**tcp_v4_connect()** initiates connection::

    int tcp_v4_connect(struct sock *sk, struct sockaddr *uaddr, int addr_len)
    {
        // Route lookup
        rt = ip_route_connect(fl4, ...);

        // Generate ISN
        tp->write_seq = secure_tcp_seq(...);

        // Send SYN
        err = tcp_connect(sk);

        // Change state
        tcp_set_state(sk, TCP_SYN_SENT);

        return err;
    }

**tcp_connect()** builds and sends SYN::

    int tcp_connect(struct sock *sk)
    {
        struct tcp_sock *tp = tcp_sk(sk);
        struct sk_buff *buff;

        // Allocate SYN packet
        buff = tcp_stream_alloc_skb(sk, ...);

        // Set SYN flag
        tcp_init_nondata_skb(buff, tp->write_seq++, TCPHDR_SYN);

        // Queue and transmit
        tcp_connect_queue_skb(sk, buff);
        tcp_transmit_skb(sk, buff, ...);

        // Start retransmit timer
        inet_csk_reset_xmit_timer(sk, ICSK_TIME_RETRANS, ...);

        return 0;
    }

**tcp_rcv_synsent_state_process()** handles SYN-ACK::

    static int tcp_rcv_synsent_state_process(struct sock *sk,
                                              struct sk_buff *skb,
                                              const struct tcphdr *th)
    {
        struct tcp_sock *tp = tcp_sk(sk);

        if (th->ack) {
            // Validate ACK
            if (!after(TCP_SKB_CB(skb)->ack_seq, tp->snd_una) ||
                after(TCP_SKB_CB(skb)->ack_seq, tp->snd_nxt))
                goto reset_and_undo;
        }

        if (th->syn) {
            // Process SYN-ACK
            tp->rcv_nxt = TCP_SKB_CB(skb)->seq + 1;
            tp->rcv_wup = TCP_SKB_CB(skb)->seq + 1;

            // Change state to ESTABLISHED
            tcp_finish_connect(sk, skb);

            // Send final ACK
            tcp_send_ack(sk);

            return -1;  // ACK sent
        }
    }


5.5 Retransmission Logic
------------------------

Retransmission is handled by **tcp_retransmit_timer()** in
``net/ipv4/tcp_timer.c:534``::

    void tcp_retransmit_timer(struct sock *sk)
    {
        struct tcp_sock *tp = tcp_sk(sk);
        struct inet_connection_sock *icsk = inet_csk(sk);
        struct sk_buff *skb;

        // Check if we have outstanding data
        if (!tp->packets_out)
            return;

        skb = tcp_rtx_queue_head(sk);

        // Check for timeout
        if (tcp_write_timeout(sk))
            goto out;

        // Enter loss state and perform retransmission
        tcp_enter_loss(sk);
        if (tcp_retransmit_skb(sk, skb, 1) > 0) {
            // Retransmit failed, try again later
            inet_csk_reset_xmit_timer(sk, ICSK_TIME_RETRANS,
                                      min(icsk->icsk_rto, TCP_RESOURCE_PROBE_INTERVAL),
                                      tcp_rto_max(sk));
            goto out;
        }

        // Update retransmit statistics
        tcp_update_rto_stats(sk);
        icsk->icsk_backoff++;
        icsk->icsk_retransmits++;

        // Restart timer with backed-off RTO
        tcp_reset_xmit_timer(sk, ICSK_TIME_RETRANS,
                             tcp_clamp_rto_to_user_timeout(sk), tcp_rto_max(sk));
    }


6. Interrupt & Concurrency Model
=================================

6.1 NAPI (New API) Architecture
-------------------------------

NAPI provides interrupt mitigation and better CPU efficiency.

**struct napi_struct** from ``include/linux/netdevice.h:379``::

    struct napi_struct {
        unsigned long       state;          /* Scheduling state */
        struct list_head    poll_list;      /* CPU poll list entry */
        int                 weight;         /* Poll budget */
        int                 (*poll)(struct napi_struct *, int);
        struct net_device   *dev;
        struct sk_buff      *skb;           /* GRO packet */
        struct gro_node     gro;
        struct hrtimer      timer;          /* Defer timer */
        struct task_struct  *thread;        /* Threaded NAPI */
        u32                 napi_id;
    };

States::

    enum {
        NAPI_STATE_SCHED,           /* Poll is scheduled */
        NAPI_STATE_MISSED,          /* Reschedule needed */
        NAPI_STATE_DISABLE,         /* Disable pending */
        NAPI_STATE_NPSVC,           /* Netpoll active */
        NAPI_STATE_THREADED,        /* Threaded polling */
    };


6.2 Interrupt-to-Softirq Flow
-----------------------------

**Step 1: Hardware Interrupt**

Driver ISR (runs with IRQs disabled on this CPU)::

    static irqreturn_t driver_interrupt(int irq, void *dev_id)
    {
        struct net_device *dev = dev_id;
        struct driver_priv *priv = netdev_priv(dev);

        // Disable further NIC interrupts
        iowrite32(0, priv->irq_enable_reg);

        // Schedule NAPI
        if (napi_schedule_prep(&priv->napi))
            __napi_schedule_irqoff(&priv->napi);

        return IRQ_HANDLED;
    }

**Step 2: Schedule Softirq**

**__napi_schedule()** in ``net/core/dev.c:6664``::

    void __napi_schedule(struct napi_struct *n)
    {
        unsigned long flags;

        local_irq_save(flags);
        ____napi_schedule(this_cpu_ptr(&softnet_data), n);
        local_irq_restore(flags);
    }

    static inline void ____napi_schedule(struct softnet_data *sd,
                                          struct napi_struct *napi)
    {
        // Add to per-CPU poll list
        list_add_tail(&napi->poll_list, &sd->poll_list);

        // Raise softirq
        __raise_softirq_irqoff(NET_RX_SOFTIRQ);
    }

**Step 3: Softirq Processing**

NET_RX_SOFTIRQ runs **net_rx_action()** (softirq context, BH)::

    static void net_rx_action(void)
    {
        struct softnet_data *sd = this_cpu_ptr(&softnet_data);
        unsigned long time_limit = jiffies + READ_ONCE(netdev_budget_usecs);
        int budget = READ_ONCE(net_hotdata.netdev_budget);

        for (;;) {
            struct napi_struct *n;

            // Get next NAPI from poll_list
            n = list_first_entry(&sd->poll_list, ...);

            // Call poll function
            budget -= napi_poll(n, &repoll);

            // Check budget
            if (budget <= 0 || time_after(jiffies, time_limit))
                break;
        }
    }

**Step 4: Driver Poll**

::

    static int driver_poll(struct napi_struct *napi, int budget)
    {
        struct driver_priv *priv = container_of(napi, ...);
        int work_done = 0;

        // Process received packets
        while (work_done < budget) {
            struct sk_buff *skb = driver_get_rx_skb(priv);
            if (!skb)
                break;

            // Deliver to network stack
            napi_gro_receive(napi, skb);
            work_done++;
        }

        if (work_done < budget) {
            // Done polling, re-enable interrupts
            napi_complete_done(napi, work_done);
            driver_enable_irq(priv);
        }

        return work_done;
    }


6.3 Socket Locking
------------------

The socket has two locks:

1. **sk_lock.slock**: Spinlock for BH protection
2. **sk_lock.owned**: Sleeping lock for user context

**lock_sock()** for process context::

    void lock_sock(struct sock *sk)
    {
        // Set owned flag, may sleep
        might_sleep();
        spin_lock_bh(&sk->sk_lock.slock);
        if (sk->sk_lock.owned)
            __lock_sock(sk);  // Sleep waiting
        sk->sk_lock.owned = 1;
        spin_unlock_bh(&sk->sk_lock.slock);
    }

**bh_lock_sock()** for softirq context::

    static inline void bh_lock_sock(struct sock *sk)
    {
        spin_lock(&sk->sk_lock.slock);
    }

**Backlog Processing**:

When softirq finds socket locked by user, packets go to backlog::

    // In tcp_v4_rcv():
    bh_lock_sock_nested(sk);
    if (!sock_owned_by_user(sk)) {
        ret = tcp_v4_do_rcv(sk, skb);
    } else {
        // Socket locked by user, queue to backlog
        sk_add_backlog(sk, skb, READ_ONCE(sk->sk_rcvbuf));
    }
    bh_unlock_sock(sk);

Backlog is processed when user releases lock::

    void release_sock(struct sock *sk)
    {
        if (sk->sk_backlog.tail)
            __release_sock(sk);  // Process backlog
        sk->sk_lock.owned = 0;
        spin_unlock_bh(&sk->sk_lock.slock);
    }


6.4 Critical Sections
---------------------

**Per-CPU softnet_data** for RX processing::

    struct softnet_data {
        struct list_head    poll_list;      /* NAPI poll list */
        struct sk_buff_head process_queue;  /* Input queue */
        unsigned int        processed;      /* Packets processed */
        struct napi_struct  backlog;        /* Per-CPU backlog NAPI */
    };

**RCU protection** for lookup tables::

    // Protocol handler lookup
    rcu_read_lock();
    ipprot = rcu_dereference(inet_protos[protocol]);
    ret = ipprot->handler(skb);
    rcu_read_unlock();


7. Error Handling & Validation
==============================

7.1 Packet Validation
---------------------

**IP Header Validation** in ``ip_rcv_core()``::

    static struct sk_buff *ip_rcv_core(struct sk_buff *skb, struct net *net)
    {
        const struct iphdr *iph;

        // Ensure we can read header
        if (!pskb_may_pull(skb, sizeof(struct iphdr)))
            goto drop;

        iph = ip_hdr(skb);

        // Version must be 4
        if (iph->version != 4)
            goto inhdr_error;

        // Header length at least 20 bytes
        if (iph->ihl < 5)
            goto inhdr_error;

        // Total length must be valid
        len = ntohs(iph->tot_len);
        if (skb->len < len)
            goto drop;

        // Verify checksum
        if (ip_fast_csum((u8 *)iph, iph->ihl))
            goto csum_error;

        return skb;

    inhdr_error:
        __IP_INC_STATS(net, IPSTATS_MIB_INHDRERRORS);
    csum_error:
        __IP_INC_STATS(net, IPSTATS_MIB_CSUMERRORS);
    drop:
        kfree_skb(skb);
        return NULL;
    }

**TCP Header Validation** in ``tcp_v4_rcv()``::

    int tcp_v4_rcv(struct sk_buff *skb)
    {
        const struct tcphdr *th;

        // Minimum header size
        if (!pskb_may_pull(skb, sizeof(struct tcphdr)))
            goto discard_it;

        th = (const struct tcphdr *)skb->data;

        // Data offset >= 5 (20 bytes)
        if (th->doff < sizeof(struct tcphdr) / 4) {
            drop_reason = SKB_DROP_REASON_PKT_TOO_SMALL;
            goto bad_packet;
        }

        // Pull full header with options
        if (!pskb_may_pull(skb, th->doff * 4))
            goto discard_it;

        // Verify checksum
        if (skb_checksum_init(skb, IPPROTO_TCP, inet_compute_pseudo))
            goto csum_error;
    }


7.2 TCP Sequence Validation
---------------------------

**tcp_validate_incoming()** in ``net/ipv4/tcp_input.c``::

    static bool tcp_validate_incoming(struct sock *sk, struct sk_buff *skb,
                                       const struct tcphdr *th, int syn_inerr)
    {
        struct tcp_sock *tp = tcp_sk(sk);

        // Sequence number validation (RFC 793)
        if (!tcp_sequence(tp, TCP_SKB_CB(skb)->seq, TCP_SKB_CB(skb)->end_seq)) {
            // Out of window
            if (!th->rst) {
                tcp_send_dupack(sk, skb);
            }
            goto discard;
        }

        // RST validation
        if (th->rst) {
            if (TCP_SKB_CB(skb)->seq == tp->rcv_nxt ||
                tcp_reset_check(sk, skb))
                return true;  // Valid RST
        }

        // SYN in established state (invalid)
        if (th->syn && !before(TCP_SKB_CB(skb)->seq, tp->rcv_nxt)) {
            tcp_send_challenge_ack(sk);
            return false;
        }

        return true;  // Packet is valid
    }


7.3 Checksum Verification
-------------------------

**IP Checksum** (verified in ip_rcv_core())::

    static inline __sum16 ip_fast_csum(const void *iph, unsigned int ihl)
    {
        // Hardware-optimized IP checksum
        return csum_fold(csum_partial(iph, ihl * 4, 0));
    }

**TCP Checksum** (hardware offload or software)::

    // Hardware indicates checksum status in skb->ip_summed
    //   CHECKSUM_UNNECESSARY: HW verified, no need to check
    //   CHECKSUM_COMPLETE: HW provided checksum, verify in SW
    //   CHECKSUM_NONE: Must compute full checksum

    // tcp_v4_rcv() calls:
    if (skb_checksum_init(skb, IPPROTO_TCP, inet_compute_pseudo))
        goto csum_error;

    // Later, tcp_checksum_complete() finalizes:
    static inline bool tcp_checksum_complete(struct sk_buff *skb)
    {
        return skb_csum_unnecessary(skb) ? 0 :
               __skb_checksum_complete(skb);
    }


7.4 Drop Reasons
----------------

The kernel tracks why packets are dropped using ``enum skb_drop_reason``::

    enum skb_drop_reason {
        SKB_DROP_REASON_NOT_SPECIFIED,
        SKB_DROP_REASON_NO_SOCKET,
        SKB_DROP_REASON_PKT_TOO_SMALL,
        SKB_DROP_REASON_TCP_CSUM,
        SKB_DROP_REASON_TCP_FLAGS,
        SKB_DROP_REASON_TCP_CLOSE,
        SKB_DROP_REASON_TCP_RESET,
        // ... many more
    };

Used throughout::

    // In tcp_v4_rcv():
    if (th->doff < sizeof(struct tcphdr) / 4) {
        drop_reason = SKB_DROP_REASON_PKT_TOO_SMALL;
        goto bad_packet;
    }

    // Final drop with reason:
    kfree_skb_reason(skb, drop_reason);


8. Performance Considerations
==============================

8.1 Hot Paths
-------------

**RX Fast Path** (tcp_rcv_established)::

    void tcp_rcv_established(struct sock *sk, struct sk_buff *skb)
    {
        // Header prediction for common case
        if ((tcp_flag_word(th) & TCP_HP_BITS) == tp->pred_flags &&
            TCP_SKB_CB(skb)->seq == tp->rcv_nxt) {
            // Fast path: in-order packet, expected flags
            __skb_queue_tail(&sk->sk_receive_queue, skb);
            sk->sk_data_ready(sk);
            return;
        }
        // Slow path...
    }

**TX Fast Path** conditions:

- Congestion window available
- Receiver window open
- No retransmissions pending
- No Nagle delay


8.2 GRO (Generic Receive Offload)
---------------------------------

GRO aggregates multiple packets before passing up::

    gro_result_t napi_gro_receive(struct napi_struct *napi,
                                   struct sk_buff *skb)
    {
        // Try to merge with existing flow
        ret = gro_receive(napi, skb);

        if (ret == GRO_HELD)
            // Packet held for merging
        else if (ret == GRO_NORMAL)
            // Pass up immediately
    }


8.3 TSO (TCP Segmentation Offload)
----------------------------------

For TX, large segments are sent to hardware for segmentation::

    // In tcp_write_xmit():
    if (skb->len > mss_now && !(TCP_SKB_CB(skb)->tcp_flags & TCPHDR_SYN))
        limit = tcp_tso_segs(sk, skb->len);

    // skb_shinfo(skb)->gso_size = mss
    // skb_shinfo(skb)->gso_type = SKB_GSO_TCPV4


8.4 NAPI Budget
---------------

NAPI limits work per poll to prevent starvation::

    // Default values
    sysctl net.core.netdev_budget = 300      // Total packets per softirq
    sysctl net.core.netdev_budget_usecs = 2000  // Time limit


8.5 Busy Polling
----------------

For latency-sensitive workloads::

    // Socket option
    setsockopt(fd, SOL_SOCKET, SO_BUSY_POLL, &val, sizeof(val));

    // Kernel polls NAPI directly in syscall context
    // instead of waiting for softirq


9. Visual Call Flow
===================

9.1 Complete RX Path
--------------------

::

    ┌─────────────────────────────────────────────────────────────────┐
    │                    PACKET ARRIVES AT NIC                        │
    └───────────────────────────────┬─────────────────────────────────┘
                                    │
                                    ▼
    ┌─────────────────────────────────────────────────────────────────┐
    │  HARDWARE INTERRUPT (IRQ Context)                               │
    │  driver_interrupt()                                             │
    │    └─► napi_schedule_prep()                                     │
    │        └─► __napi_schedule_irqoff()                             │
    │            └─► ____napi_schedule()                              │
    │                └─► __raise_softirq_irqoff(NET_RX_SOFTIRQ)       │
    └───────────────────────────────┬─────────────────────────────────┘
                                    │
                                    ▼
    ┌─────────────────────────────────────────────────────────────────┐
    │  SOFTIRQ (Softirq Context / NET_RX_SOFTIRQ)                     │
    │  net_rx_action()                                                │
    │    └─► napi_poll()                                              │
    │        └─► __napi_poll()                                        │
    │            └─► driver->poll()                                   │
    │                └─► napi_gro_receive()                           │
    │                    └─► napi_skb_finish()                        │
    │                        └─► netif_receive_skb_internal()         │
    └───────────────────────────────┬─────────────────────────────────┘
                                    │
                                    ▼
    ┌─────────────────────────────────────────────────────────────────┐
    │  CORE NETWORK (net/core/dev.c)                                  │
    │  __netif_receive_skb()                                          │
    │    └─► __netif_receive_skb_core()                               │
    │        ├─► XDP processing (if enabled)                          │
    │        ├─► VLAN handling                                        │
    │        ├─► TC ingress                                           │
    │        ├─► rx_handler (bridging/bonding)                        │
    │        └─► deliver_skb() [based on skb->protocol]               │
    │            └─► ptype->func() → ip_rcv()                         │
    └───────────────────────────────┬─────────────────────────────────┘
                                    │
                                    ▼
    ┌─────────────────────────────────────────────────────────────────┐
    │  IP LAYER (net/ipv4/ip_input.c)                                 │
    │  ip_rcv()                                                       │
    │    └─► ip_rcv_core() [validate IP header]                       │
    │        └─► NF_HOOK(PRE_ROUTING)                                 │
    │            └─► ip_rcv_finish()                                  │
    │                └─► ip_route_input_noref() [routing decision]    │
    │                    └─► ip_local_deliver() [for local delivery]  │
    │                        └─► NF_HOOK(LOCAL_IN)                    │
    │                            └─► ip_local_deliver_finish()        │
    │                                └─► ip_protocol_deliver_rcu()    │
    │                                    └─► tcp_v4_rcv()             │
    └───────────────────────────────┬─────────────────────────────────┘
                                    │
                                    ▼
    ┌─────────────────────────────────────────────────────────────────┐
    │  TCP LAYER (net/ipv4/tcp_ipv4.c, tcp_input.c)                   │
    │  tcp_v4_rcv()                                                   │
    │    ├─► validate TCP header & checksum                           │
    │    ├─► __inet_lookup_skb() [find socket]                        │
    │    ├─► bh_lock_sock()                                           │
    │    └─► tcp_v4_do_rcv()                                          │
    │        ├─► [ESTABLISHED] tcp_rcv_established() [FAST PATH]      │
    │        │   ├─► tcp_ack()                                        │
    │        │   ├─► tcp_data_queue()                                 │
    │        │   └─► sk->sk_data_ready()                              │
    │        └─► [OTHER STATES] tcp_rcv_state_process()               │
    └───────────────────────────────┬─────────────────────────────────┘
                                    │
                                    ▼
    ┌─────────────────────────────────────────────────────────────────┐
    │  SOCKET LAYER                                                   │
    │  sk->sk_data_ready()                                            │
    │    └─► sock_def_readable()                                      │
    │        └─► wake_up_interruptible_sync_poll()                    │
    └───────────────────────────────┬─────────────────────────────────┘
                                    │
                                    ▼
    ┌─────────────────────────────────────────────────────────────────┐
    │  APPLICATION (User Context)                                     │
    │  read()/recv()/recvmsg()                                        │
    │    └─► sock->ops->recvmsg()                                     │
    │        └─► tcp_recvmsg()                                        │
    │            └─► skb_copy_datagram_msg() [copy to user]           │
    └─────────────────────────────────────────────────────────────────┘


9.2 Complete TX Path
--------------------

::

    ┌─────────────────────────────────────────────────────────────────┐
    │  APPLICATION (User Context)                                     │
    │  write()/send()/sendmsg()                                       │
    │    └─► sock->ops->sendmsg()                                     │
    │        └─► tcp_sendmsg()                                        │
    └───────────────────────────────┬─────────────────────────────────┘
                                    │
                                    ▼
    ┌─────────────────────────────────────────────────────────────────┐
    │  TCP LAYER (net/ipv4/tcp.c, tcp_output.c)                       │
    │  tcp_sendmsg_locked()                                           │
    │    ├─► tcp_stream_alloc_skb()                                   │
    │    ├─► skb_add_data_nocache() [copy from user]                  │
    │    └─► tcp_push()                                               │
    │        └─► __tcp_push_pending_frames()                          │
    │            └─► tcp_write_xmit()                                 │
    │                ├─► tcp_cwnd_test() [congestion control]         │
    │                ├─► tcp_snd_wnd_test() [receiver window]         │
    │                ├─► tcp_tso_segs() [TSO segmentation]            │
    │                └─► tcp_transmit_skb()                           │
    │                    ├─► Build TCP header                         │
    │                    ├─► tcp_options_write()                      │
    │                    ├─► tcp_v4_send_check() [checksum]           │
    │                    └─► ip_queue_xmit()                          │
    └───────────────────────────────┬─────────────────────────────────┘
                                    │
                                    ▼
    ┌─────────────────────────────────────────────────────────────────┐
    │  IP LAYER (net/ipv4/ip_output.c)                                │
    │  ip_queue_xmit()                                                │
    │    └─► __ip_queue_xmit()                                        │
    │        ├─► ip_route_output_ports() [routing]                    │
    │        ├─► Build IP header                                      │
    │        └─► ip_local_out()                                       │
    │            ├─► ip_send_check() [IP checksum]                    │
    │            └─► NF_HOOK(LOCAL_OUT)                               │
    │                └─► dst_output()                                 │
    │                    └─► ip_output()                              │
    │                        └─► NF_HOOK(POST_ROUTING)                │
    │                            └─► ip_finish_output()               │
    │                                └─► ip_finish_output2()          │
    │                                    └─► neigh_output()           │
    │                                        └─► dev_queue_xmit()     │
    └───────────────────────────────┬─────────────────────────────────┘
                                    │
                                    ▼
    ┌─────────────────────────────────────────────────────────────────┐
    │  CORE NETWORK (net/core/dev.c)                                  │
    │  __dev_queue_xmit()                                             │
    │    ├─► netdev_core_pick_tx() [select TX queue]                  │
    │    ├─► sch_handle_egress() [TC egress]                          │
    │    └─► __dev_xmit_skb() or dev_hard_start_xmit()                │
    │        └─► dev_hard_start_xmit()                                │
    │            └─► xmit_one()                                       │
    │                └─► netdev_start_xmit()                          │
    │                    └─► dev->netdev_ops->ndo_start_xmit()        │
    └───────────────────────────────┬─────────────────────────────────┘
                                    │
                                    ▼
    ┌─────────────────────────────────────────────────────────────────┐
    │  DRIVER LAYER                                                   │
    │  driver_ndo_start_xmit()                                        │
    │    ├─► Map skb to DMA                                           │
    │    ├─► Write to TX ring buffer                                  │
    │    └─► Ring doorbell (notify hardware)                          │
    └───────────────────────────────┬─────────────────────────────────┘
                                    │
                                    ▼
    ┌─────────────────────────────────────────────────────────────────┐
    │  HARDWARE                                                       │
    │  NIC fetches packet via DMA and transmits on wire               │
    └─────────────────────────────────────────────────────────────────┘


9.3 TCP Connection Establishment
--------------------------------

::

    Client (TCP_CLOSED)                           Server (TCP_LISTEN)
         │                                              │
         │  tcp_v4_connect()                            │
         │  tcp_connect()                               │
         │    └─► send SYN                              │
         ├──────────────────── SYN ────────────────────►│
         │                                              │  tcp_v4_rcv()
         │  tcp_set_state(TCP_SYN_SENT)                 │  tcp_rcv_state_process()
         │                                              │    └─► tcp_conn_request()
         │                                              │        └─► send SYN-ACK
         │◄─────────────────── SYN-ACK ─────────────────┤
         │  tcp_rcv_synsent_state_process()             │  inet_csk_reqsk_queue_hash_add()
         │    └─► tcp_finish_connect()                  │  (TCP_NEW_SYN_RECV mini-sock)
         │        └─► send ACK                          │
         │  tcp_set_state(TCP_ESTABLISHED)              │
         ├──────────────────── ACK ────────────────────►│
         │                                              │  tcp_v4_rcv()
         │                                              │    └─► tcp_check_req()
         │                                              │        └─► tcp_create_openreq_child()
         │                                              │  tcp_set_state(TCP_ESTABLISHED)
         │                                              │  inet_csk_complete_hashdance()
         ▼                                              ▼
    (TCP_ESTABLISHED)                             (TCP_ESTABLISHED)


10. Summary
===========

This document traced a TCP packet's journey through the Linux kernel:

**RX Path Key Functions:**

1. ``driver_interrupt()`` → ``napi_schedule()``
2. ``net_rx_action()`` → ``driver_poll()``
3. ``netif_receive_skb()`` → ``__netif_receive_skb_core()``
4. ``ip_rcv()`` → ``ip_local_deliver()``
5. ``tcp_v4_rcv()`` → ``tcp_rcv_established()``
6. ``sk->sk_data_ready()`` → application wakes

**TX Path Key Functions:**

1. ``tcp_sendmsg()`` → ``tcp_sendmsg_locked()``
2. ``tcp_write_xmit()`` → ``tcp_transmit_skb()``
3. ``ip_queue_xmit()`` → ``ip_local_out()``
4. ``dev_queue_xmit()`` → ``dev_hard_start_xmit()``
5. ``driver->ndo_start_xmit()`` → hardware

**Key Data Structures:**

- ``struct sk_buff``: Packet buffer (include/linux/skbuff.h)
- ``struct sock``: Socket (include/net/sock.h)
- ``struct tcp_sock``: TCP-specific socket (include/linux/tcp.h)
- ``struct napi_struct``: NAPI polling (include/linux/netdevice.h)
- ``struct net_device``: Network interface (include/linux/netdevice.h)

**Key Files:**

- ``net/core/dev.c``: Core network device handling
- ``net/core/skbuff.c``: Socket buffer operations
- ``net/ipv4/ip_input.c``: IP receive path
- ``net/ipv4/ip_output.c``: IP transmit path
- ``net/ipv4/tcp_input.c``: TCP receive processing
- ``net/ipv4/tcp_output.c``: TCP transmit processing
- ``net/ipv4/tcp_ipv4.c``: IPv4-specific TCP
