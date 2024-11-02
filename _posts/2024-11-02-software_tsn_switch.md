---
title:  "TSN on the Edge: Software TSN Switch for the Edge Cloud"
date:   2024-11-02 00:00:00 +0100
categories: Linux TSN networking edge cloud
tags: Linux TSN networking edge cloud
authors :
  - Frank Duerr
---
In this blog post, I am going to explain how to set up a software TSN (Time-sensitive Networking) switch with Linux supporting the Time-aware Shaper from the IEEE 802.1Q standard (formerly IEEE 802.1Qbv).

A disclaimer first: TSN came with the goal of enabling 'hard' real-time communication with deterministic bounds on network delay and jitter over standard Ethernet (IEEE 802.3 networks). Obviously, it can be debated to which extent a software switch and the Linux kernel can support such deterministic bounds. You have to decide yourself whether you want to trust a software TSN switch with the control of your industrial plant, networked car, nuclear plant, or communication within your space station. But seriously, I think software TSN switches can be useful even without hard guarantees, be it for teaching students TSN in your network lab, doing experiments without investing into expensive hardware, or researching how to actually extend the current implementation to finally really provide deterministic guarantees. In some respects, software switches even have inherent advantages, in particular, if it comes to flexibility. We will see that later when it comes to classifying packets using information from all network layers (not just the data link layer where Ethernet resides). Even combining TSN and SDN (Software-defined Networking) now seems to be a matter of 'plumbing' together an SDN switch and a suitable Linux queuing discipline for TSN.

If you have never heard of TSN or the Time-aware Shaper, you should start reading the following TSN background section first, where I give a brief overview of the Time-aware Shaper. All TSN experts can safely skip this section and directly jump to the description of the Time-aware Priority Shaper (TAPRIO), the new Linux queuing discipline implementing the Time-aware Shaper. Finally, I will show how to integrate a Linux bridge, iptables classifier, and the Time-aware Priority Shaper into a software TSN switch.

# TSN Background: Time-aware Shaper

Time-sensitive Networking (TSN) is a collection of IEEE standards to enable real-time communication over IEEE 802.3 networks (Ethernet). Although several implementations of real-time Ethernet technologies have already existed for some time in the past, TSN now brings real-time communication to standard Ethernet as defined by IEEE. With TSN, a TSN-enabled Ethernet can now transport both, real-time and non-real-time traffic over one converged network.

At the center of the TSN standards are different so-called shapers, which some people would call schedulers, and others queuing disciplines, so don’t be confused if I use these words interchangeably. Deterministic real-time communication with very low delay and jitter is the realm of the so-called **Time-aware Shaper (TAS)**. Basically, the TAS implements a TDMA scheme, by giving packets (or frames as they are called on the data link layer) of different traffic classes access to the medium within different time slots. To understand the technical details better, let’s have a look at how a packet traverses a switch. The following figure shows a simplified but sufficiently accurate view onto the data path of a TSN switch.

```
          incoming packet (from driver/NIC)
                             |
                             v
+------------------------------------------------------------+
|                      Forwarding Logic                      +
+------------------------------------------------------------+
               | output on port 1   ...    | output on port n       
               v                           v
+--------------------------------+
+          Classifier            +
+--------------------------------+
    |          |             |
    v          v             v
+-------+  +-------+     +-------+  
|       |  |       |     |       |
| Queue |  | Queue | ... | Queue |
|  TC0  |  |  TC1  |     |  TC7  |  
|       |  |       |     |       |
+-------+  +-------+     +-------+
    |          |             |         +-------------------+ 
    v          v             v         | Gate Control List |
+-------+  +-------+     +-------+     | t1 10000000       |  
| Gate  |<-| Gate  | ... | Gate  |<----| t2 01111111       |
+-------+  +-------+     +-------+     | ...               |
    |          |             |         | repeat            |
    v          v             v         +-------------------+
+--------------------------------+     
|     Transmission Selection     |     
+--------------------------------+
               |
	       v
        to driver/NIC 
```

First, the packets enters the switch through the incoming port or the network interface controller (NIC) of your Linux box implementing the software switch. Then, the forwarding logic decides on which outgoing port to forward the packet. So far, this is not different from an ordinary switch.

Then comes the more interesting part from the point of view of a TSN switch. For the following discussion, we zoom into one outgoing port (this part of the figure should be replicated n times, once for each outgoing port). First, the classifier decides, which **traffic class** the packet belongs to. To this end, the VLAN tag of the packet contains a three-bit Priority Code Point (PCP) field. So it should not come as a big surprise that eight different traffic classes are supported, each having its own outgoing FIFO queue, i.e., eight queues per outgoing port.

Now comes the big time (no pun intended) of the Time-aware Shaper (TAS): Behind each queue is a gate. If the gate of a queue is open, the first packet in the queue is eligible for transmission. If the gate is closed, the queue cannot transmit. Whether a gate is open or closed is defined by the time schedule stored in the Gate Control List (GCL). Each entry in the GCL has a timestamp defining the time when the state of the gates should change to a given state. For instance the entry `t1 10000000` says that at time t1 gate 0 should be open (1) and gates 1-7 should be closed (0). After the end of the schedule, the schedule repeats in a cyclic fashion, i.e., t1, t2, etc. define relative times with respect to the start of a cycle. For a defined behavior, the clocks of all switches need to be synchronized, so all switches refer to the same cycle base time with their schedules. This is the job of the PTP (Precision Time Protocol).

The idea is that gates along the path of a time-sensitive packet are opened and closed such that an upper bound on network delay and jitter can be guarateed despite concurrent traffic, which might need to wait behind closed gates. How to calculate time schedules to guaranteed a desired upper bound on network delay and jitter is out of the scope of the IEEE standard. Actually, it's a hard problem and subject to active research. I will not got into detail here, but just mention that also we have defined algorithms for calculating TAS schedules as part of our research at University of Stuttgart (e.g. this [paper](https://dl.acm.org/doi/10.1145/2997465.2997494)) as well as others (a recent survey from researcher of University of Tübingen with further references can be found [here](https://arxiv.org/abs/2211.10954)).

Ok, well enough, but what happens if two gates are open at the same time? Yes, that's allowed, as the schedule entry `t2 011111111` shows where 7 gates are open all at the same time. Then Transmission Selection will refer to a second scheduling algorithm, e.g., strict priority queuing to decided which packet from a queue with open gate is allowed to transmit. You see, several scheduling mechanisms work together here, and I did not even mention the other IEEE shapers such as the Credit-based Shaper, which could be added to this picture. Here, we just focus on the TAS, keeping in mind that also many hardware switches might not implement all possible shapers defined by IEEE.

# The Linux Time-aware Priority Shaper

The Time-aware Shaper as defined by IEEE standards introduced in the previous section is implemented by the Linux queuing descipline Time-aware Priority Shaper (TAPRIO). So let's see how to configure TAPRIO for a network interface.

Configuration of queuing disciplines (or QDISCs for short) is done with the tc (traffic control) tool. Let’s assume that we want to set up TAPRIO for all traffic leaving through the network interface enp2s0f1. Then, the tc command could look as follows:

```
$ tc qdisc replace dev enp2s0f1 parent root handle 100 taprio \
num_tc 2 \
map 1 0 1 1 1 1 1 1 1 1 1 1 1 1 1 1 \
queues 1@0 1@1 \
base-time 1554445635681310809 \
sched-entry S 01 800000 sched-entry S 02 200000 \
clockid CLOCK_TAI
```

Here, we replace the existing QDISC (maybe the default one) of the device enp2s0f1 by a TAPRIO QDISC, which is placed right at the root of the device. We need to provide a unique handle (100) for this QDISC.

We define two traffic classes (`num_tc 2`). As you have seen above, an IEEE switch might have queues for up to 8 traffic classes. TAPRIO supports up to 16 traffic classes, although your NIC then also would need as many TX queues (see below).

Then, we need to define how to classify packets, i.e., how to assign packets to traffic classes. To this end, TAPRIO uses the priority field of the sk_buff (socket buffer, SKB) structure. The SKB is the internal kernel data structure for managing packets. Since the SKB is a kernel structure, you cannot directly set it from user space. One way of setting it from user space is to use the SO_PRIORITY socket option by the sending application. However, since we are implementing a switch here, the sending application might reside on another host, so for our use case this is not an option. As described below, we will use another possibility, namely iptables, to set the priority field of SKB before they reach the QDISC. For now, let’s assume the priority is set somehow. Then, the map parameter defines the mapping of SKB priority values to traffic classes (TC) using a bit vector. You can read the bit vector `1 0 1 1 1 1 1 1 1 1 1 1 1 1 1 1` as follows: map priority 0 (first bit from the left) to TC1, priority 1 to TC0, and priorities 2-15 to TC1 (16 mappings for 16 possible traffic classes).

Next, we map traffic classes to TX queues of the network device. Modern network devices typically implement more than one TX queue for outgoing traffic. How many TX queues are supported by your device, you can find out with the following command:

```
$ ls /sys/class/net/enp2s0f1/queues/
rx-0 rx-1 rx-2 rx-3 rx-4 rx-5 rx-6 rx-7 tx-0 tx-1 tx-2 tx-3 tx-4 tx-5 tx-6 tx-7
```

Here, the network device enp2s0f1 supports 8 TX queues, more than enough for our two traffic classes. The parameter `queues 1@0 1@1` reads like this: The first entry `1@0` defines the mapping of the first traffic class (TC0) to TX queues, the second entry `1@1` the mapping of the second traffic class (TC1), and so on. Each entry defines a range of queues to which the traffic class should be mapped using the schema queue_count@queue_offset. That is, `1@0` means map (the traffic class) to 1 TX queue starting at queue index 0, i.e., queue range [0,0]. The second class is also mapped to 1 queue at queue index 1 (`1@1`). You can also map one traffic class to several TX queues by increasing the count parameter beyond 1. Make sure that queue ranges do not overlap.

Next, we define the schedule of the Time-aware Shaper implemented by TAPRIO. First of all, we need to define a base time as a reference for the cyclic schedule. Every scheduling cycle starts at base_time + k*cycle_time. The cycle time (duration of the cycle until it repeats) is implicitly defined by the sum of the times (interval durations) of the schedule entries (see below), in our example 800000 ns + 200000 ns = 1000000 ns = 1 ms. The base time is defined in nano seconds according to some clock. The reference clock to be used is defined by parameter clockid. CLOCK_TAI is the International Atomic Time. The advantages of TAI are: TAI is not adjusted by leap seconds in contrast to CLOCK_REALTIME, and TAI refers to a well-defined starting time in contrast to CLOCK_MONOTONIC.

Finally, we need to define the entries of the Gate Control List, i.e., the points in time when gates should open or close (or in other words: the time intervals during which gates are open or closed). For instance, `sched-entry S 01 800000` says that the gate of TC0 (least significant bit in bit vector) opens at the start of the cycle for 800000 ns duration, and all other gates are closed for this interval. Then, 800000 ns after the start of the cycle, the entry `sched-entry S 02 200000` defines that the gate of TC1 (second bit of bit vector) opens for 200000 ns, and all other gates are closed.

Note that as said above, you can also open multiple gates at the same time by setting multiple bits of the bit vector. Now, the transmission selection algorithm should decide which packet from one of the queues with open gate to send next. The manual page of TAPRIO does not clearly say, which open queue gets priority. However, from looking at the source code of TAPRIO, it seems that TAPRIO gives open queues with smaller queue number priority.

# Software TSN Switch

Now that we know how to use the TAPRIO QDISC, we can finally set up our software TSN switch. The software TSN switch integrates three parts:

* A software switch (aka bridge) taking care of forwarding packets to the right outgoing port.
* Per traffic class a classifier implemented through iptables defining the priority of forwarded packets, which is then mapped to a corresponding traffic class by TAPRIO.
* Per outgoing switch port (network interface) a TAPRIO QDISC.

First, we set up a software switch (aka bridge) called br0:

```
$ brctl addbr br0
```

We assign two network interfaces (enp2s0f0 and enp2s0f1) to the switch, which we first put into promiscuous mode, so the switch will see all incoming packets:

```
$ ip link set dev enp2s0f0 promisc on
$ ip link set dev enp2s0f1 promisc on
```

Then, we assign the two interfaces to the switch:

```
$ brctl addif br0 enp2s0f0
$ brctl addif br0 enp2s0f1
```

Finally, we bring the switch up:

```
$ ip link set dev br0 up
```

Next, we define classifiers for each traffic class using iptables. As said above, TAPRIO uses the SKB priority field to map priorities to traffic classes. Assume that we want to assign priority 1 to all UDP packets with destination port 6666. With iptables, we can implement a corresponding classifier rule like this:

```
$ iptables -t mangle -A POSTROUTING -p udp --dport 6666 -j CLASSIFY --set-class 0:1
```

The mangle table (`-t mangle`) is used to modify packets, so this is what we need here. The priority field of the SKB can be set using the argument `-j CLASSIFY` together with the argument `–set-class 0:1` to define the priority value (here 0:1, i.e., 1). The argument name `set-class` might sound confusing because the mapping of priorities to traffic classes is actually done by the TAPRIO QDISC. This value is actually the value for the priority field of the SKB. The argument `-A POSTROUTING` appends a rule to the POSTROUTING chain, which is invoked after the forwarding decision, just before the packet reaches the QDISC, so the QDISC can see the priority field set by the iptables rule (classifier). UDP packets can be matched by the protocol argument `-p`, and the destination port by the `–dport` argument. For each traffic class, you need to set up a corresponding classifier rule.

One more thing to note is that bridges typically work on layer 2 (data link layer), whereas iptables typically deal with higher layers (network layer and transport layer). Since kernel 2.6, bridged traffic can be exposed to iptables through bridge-nf. So we need to enable forwarding bridged IPv4 traffic and IPv6 traffic to iptables by setting the following sysctl entries to 1:

```
$ echo 1 > /proc/sys/net/bridge/bridge-nf-call-iptables
$ echo 1 > /proc/sys/net/bridge/bridge-nf-call-ip6tables
```

If you are missing these sysctl entries, be sure to load the br_netfilter module first:

```
$ modprobe br_netfilter
```

Note that iptables is pretty versatile in classifying packets since we can use information from different protocol layers, not just layer 2. If you actually want to match on layer 2 information such as the PCP field of the VLAN tag (as typically done by pure layer 2 TSN switches), we need to make VLAN tag information visible to iptables using the following command:

```
$ echo 1 > /proc/sys/net/bridge/bridge-nf-filter-vlan-tagged
```

Then, you can match the bits of the PCP using the u32 module of iptables (I didn’t actually check this, so I hope I am not off by some bits):

```
$ iptables -t mangle -A POSTROUTING -m u32 --u32 "12&0x0000E000=0x0000200" -j CLASSIFY --set-class 0:1
```

`12&0x0000E000` specifies the bits to match, where 12 is the byte offset from the beginning of the frame (starting with 0 for the first byte), and 0x0000E000 is the mask applied to the matched 4 bytes. In the Ethernet frame, the VLAN tag is preceeded by 2×6 bytes for the destination MAC address and source MAC address, respectively. Thus, the offset of the VLAN tag is 12 bytes. The PCP field are the 3 most significant bits of the 3rd byte of the VLAN tag. Thus, the mask is 0x0000E000. 0x0000200 is the PCP value 1. If you plan to use the PCP field for classification, you need to ensure that all packets are VLAN-tagged since this rule matches on raw bits without checking whether the packets is actually VLAN-tagged.

Finally, for each port, set up the TAPRIO qdisc responsible for time-aware scheduling of outgoing traffic of that port as already shown above.

# A Small Test

Finally, we can test our software TSN switch in a little scenario with a single two-port TSN software switch. The switch has two physical 1 GE ports. We have two senders, each sending a stream of UDP packets to a different receiver process, i.e., we have two flows (the red and the blue flow). Both senders reside on the same host attached to switch port #1. Both receivers reside on another host attached to switch port #2.

On port #2 (towards the receivers), we set up a TAPRIO QDISC with 800 us time slot for the blue flow (traffic class 0, TC0) and 200 us time slot for the red flow (traffic class 1, TC1):

```
$ tc qdisc replace dev enp2s0f1 parent root handle 100 taprio \
num_tc 2 \
map 1 0 1 1 1 1 1 1 1 1 1 1 1 1 1 1 \
queues 1@0 1@1 \
base-time 1554445635681310809 \
sched-entry S 01 800000 sched-entry S 02 200000 \
clockid CLOCK_TAI
```

Two classifiers are set up to classify red traffic to port 7777 as TC1 mapped from priority 0 (0:0), and blue traffic to port 6666 as TC0 mapped from priority 1 (0:1):

```
$ iptables -t mangle -A POSTROUTING -p udp --dport 6666 -j CLASSIFY --set-class 0:1
$ iptables -t mangle -A POSTROUTING -p udp --dport 7777 -j CLASSIFY --set-class 0:0
```

Both senders send as fast as possible. At the receiver side, incoming packets are timestamped using the hardware timestamping feature of the NIC, thus, recorded time stamps should be pretty accurate.

The following figure shows the arrival times of packets at the receivers. We draw a vertical line whenever a packet of the red or blue flow is received (the individual lines blend together at higher data rates). As we can see, the packets of the two flows arrive nicely separated within their assigned time slots of duration 800 us and 200 us, respectively. After 1 ms, the cycle repeats. So time-aware shaping works!

![Time-shaped traffic]({{ site.url }}{{ site.baseurl }}/assets/images/taprio-traffic.png "Time-shaped traffic"){: .align-center}

We can also see that right at the start of each time slot, packets arrive at maximum rate (1 Gbps), whereas later in a time slot, the rate is lower (see blue flow). The reason for this is that right after the gate of a queue opens, queued packets are forwarded over the outgoing interface (port) at full line rate. However, since in our scenario both flows share the same incoming (bottleneck) link into the switch, both flows only have an average data rate of 500 Mbps each into the switch, thus, the data rate on the outgoing link drops, when the open queue runs empty. Note that the time-aware shaper is shaping the outgoing traffic, not the incoming traffic. So this is not a problem of the switch, but due to the setup.

# Where to go from here

In this tutorial, we configured a simple software TSN switch. One interesting extension could be to configure an SDN+TSN-Switch by replacing the Linux bridge by an Open vSwitch. Maybe, this will be part of another tutorial.
