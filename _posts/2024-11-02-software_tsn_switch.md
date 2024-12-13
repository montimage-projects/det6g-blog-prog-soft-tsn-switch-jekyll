---
title:  "TSN on the Edge: Software TSN Bridge for the Edge Cloud"
date:   2024-11-02 00:00:00 +0100
categories: Linux TSN networking edge cloud
tags: Linux TSN networking edge cloud
authors :
  - Frank Duerr
  - Jose Costa-Requena
---
In this blog post, we explain how to set up a software TSN bridge implementing time-aware shaping with Linux on an edge cloud server.

# tl;dr -- Takeaway Messages

* Many networked real-time systems utilize edge computing servers to offload computations.
* Time-Sensitive Networking (TSN) is one technology to support deterministic real-time communication with its Time-Aware Shaper to communicate with edge cloud servers.
* Using virtualization technologies like containers or virtual machines (VM) on edge servers requires software bridges to connect containers, VMs, etc. The TSN network effectively extends onto the edge server. 
* The Time-Aware Priority Shaper (TAPRIO) is one technology to implement the Time-Aware Shaper on Linux software bridges.
* In this blog post, we explain and demonstrate how to combine TAPRIO and Linux virtual bridges into software TSN bridges. 

# Motivation

Networked real-time system often utilize an edge cloud computing environment to execute components on edge cloud servers. A prominent example are networked control systems, where the controller of the control systems is offloaded to an edge cloud server that communicates with the 'plant' consisting of sensors and actuators over the network as shown in the following figure.

![Networked control system]({{ site.url }}{{ site.baseurl }}/assets/images/networked_control_system.png "Networked control system"){: .align-center}

The DETERMINISTIC6G project describes a number of such applications in [this document](https://deterministic6g.eu/images/deliverables/DETERMINISTIC6G-D1.1-v1.0.pdf), including:

* Automated guided vehicles moving on a shop floor in a factory and communicating with edge cloud servers in the factory.
* Exoskeletons assisting workers on a shop floor, which are remotely controlled from an edge cloud server in the factory.
* Extended reality devices like Augmented Reality (AR) headsets offloading compute-intensive tasks like rendering of images to an edge server.

Real-time communication technologies are used to guarantee bounds on the network delay between plant and controller. Time-Sensitive Networking (TSN) is a popular technology to implement real-time communication over IEEE 802.3 (Ethernet) networks consisting of TSN bridges. In particular, the so-called Time-Aware Shaper (TAS) implemented by bridges is able to guarantee very low bounds on network delay and delay variation (jitter).

Today, edge cloud servers often utilize virtualization technologies such as virtual machines (VM) or containers to host and isolate application components such as the controller of a networked control system as shown in the figure above. Typically, VMs or containers are connected to the physical network through a software bridge. That is, the network does not end at the network interface of the host, but extends onto the host up to the network interfaces of the VMs or containers, including in particular the software bridge. Consequently, real-time end-to-end communication must also include the software bridge. In particluar, similarly to the hardware bridges, the software bridge should also implement the TAS.

The Linux operating system comes with an implementation of the TAS called the Time-Aware Priority Shaper (TAPRIO) which can be combined with the Linux software bridge to implement a software TSN bridge. However, the usage of TAPRIO and in particular the combination of TAPRIO and software bridges is not trivial. Therefore, the goal of this blog post is to explain how to implement a software TSN bridge with TAPRIO with Linux.  

If you have never heard of TSN or the Time-aware Shaper, you should start reading the following TSN background section first, where we give a brief overview of the TAS. All TSN experts can safely skip this section and directly jump to the description of TAPRIO, the Linux queuing discipline implementing the Time-aware Shaper. Finally, we will show how to integrate a Linux bridge with TAPRIO into a software TSN bridge.

# TSN Background: Time-aware Shaper for Scheduled Traffic

Time-sensitive Networking (TSN) is a collection of IEEE standards to enable real-time communication over IEEE 802.3 networks (Ethernet). Although several implementations of real-time Ethernet technologies have already existed for some time in the past, TSN now brings real-time communication to standard Ethernet as defined by IEEE. With TSN, a TSN-enabled Ethernet can now transport both, real-time and non-real-time traffic over one converged network.

At the center of the TSN standards are different so-called shapers, which some people would call schedulers, and others queuing disciplines, so don’t be confused if we use these words interchangeably. Deterministic real-time communication with very low delay and jitter is the realm of the so-called Time-aware Shaper (TAS). Basically, the TAS implements a TDMA scheme, by giving packets (or frames as they are called on the data link layer) of different traffic classes access to the medium within different time slots. To understand the technical details better, let’s have a look at how a packet traverses a bridge. The following figure shows a simplified but sufficiently accurate view onto the data path of a TSN bridge.

```
          incoming packet from ingress network interface
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
outgoing packet to egress network interface 
```

First, the packets enters the bridge through the incoming port or the network interface controller (NIC). Then, the forwarding logic decides on which outgoing port to forward the packet. So far, this is not different from an ordinary bridge.

Then comes the more interesting part from the point of view of a TSN switch. For the following discussion, we zoom into one outgoing port (this part of the figure should be replicated n times, once for each outgoing port). First, the classifier decides, which traffic class the packet belongs to. To this end, the VLAN tag of the packet contains a three-bit Priority Code Point (PCP) field. So it should not come as a big surprise that eight different traffic classes are supported, each having its own outgoing FIFO queue, i.e., eight queues per outgoing port.

The TAS controls when packets from which class are eligible for forwarding. Behind each queue is a gate. If the gate of a queue is open, the first packet in the queue is eligible for transmission. If the gate is closed, the queue cannot transmit. Whether a gate is open or closed is defined by the time schedule stored in the Gate Control List (GCL). Each entry in the GCL has a timestamp defining the time when the state of the gates should change to a given state. For instance the entry `t1 10000000` says that at time t1 gate 0 should be open (1) and gates 1-7 should be closed (0). After the end of the schedule, the schedule repeats in a cyclic fashion, i.e., t1, t2, etc. define relative times with respect to the start of a cycle. For a defined behavior, the clocks of all switches need to be synchronized, so all switches refer to the same cycle base time with their schedules. This is the job of the PTP (Precision Time Protocol).

The idea is that gates along the path of a time-sensitive packet are opened and closed such that upper bounds on the end-to-end network delay and jitter can be guarateed despite concurrent traffic, which might need to wait behind closed gates. How to calculate schedules to guaranteed a desired upper bound on end-to-end network delay and jitter is out of the scope of the IEEE standard. Actually, it is a hard problem and subject to active research. We will not got into detail here, but refer the interested reader to a [recent survey](https://arxiv.org/abs/2211.10954)).

It is also allowed to open multiple gates at the same time. For instance, the scheduling entry `t2 011111111` would open 7 gates all at the same time. Then Transmission Selection will decided which queue with an open gate is allowed to transmit a packet, e.g., using strict priority queuing.

# The Linux Time-aware Priority Shaper

The Time-aware Shaper (TAS) as defined by IEEE standards introduced in the previous section is implemented by the Linux Queuing Discipline (QDisc) Time-aware Priority Shaper (TAPRIO). QDiscs are a powerful Linux concept to arrange packets to be output over a network interface to which the QDisc is attached. You can even define chains of QDiscs that a packets passes through on its way to the egress interface. Next, we describe briefly how to configure TAPRIO for a network interface.

The configuration of a QDisc is done with the tc (traffic control) tool. Let’s assume that we want to set up TAPRIO for all traffic leaving through the network interface enp2s0f1. Then, the tc command could look as follows:

```
$ tc qdisc replace dev enp2s0f1 parent root handle 100 taprio \
num_tc 2 \
map 1 0 1 1 1 1 1 1 1 1 1 1 1 1 1 1 \
queues 1@0 1@1 \
base-time 1554445635681310809 \
sched-entry S 01 800000 sched-entry S 02 200000 \
clockid CLOCK_TAI
```

Here, we replace the existing QDisc (maybe the default one) of the device enp2s0f1 by a TAPRIO QDisc, which is placed right at the root of the device. We need to provide a unique handle (100) for this QDisc.

We define two traffic classes (`num_tc 2`). As you have seen above, an IEEE switch might have queues for up to 8 traffic classes. TAPRIO supports up to 16 traffic classes, although your NIC then also would need as many transmit (TX) queues (see below).

Then, we need to define how to classify packets, i.e., how to assign packets to traffic classes. To this end, TAPRIO uses the priority field of the sk_buff structure (socket buffer, SKB for short). The SKB is the internal kernel data structure for managing packets. Since the SKB is a kernel structure, you cannot directly set it from user space. One way of setting the priority field of an SKP from user space is to use the SO_PRIORITY socket option by the sending application. Another option, which is particularly important for a bridge, is to map the PCP value from a packet to the SKB priority as shown further below. For now, let’s assume the priority is set somehow. Then, the map parameter defines the mapping of SKB priority values to traffic classes (TC) using a bit vector (positional parameters). You can read the bit vector `1 0 1 1 1 1 1 1 1 1 1 1 1 1 1 1` as follows: map priority 0 (first bit from the left) to TC1, priority 1 to TC0, and priorities 2-15 to TC1 (16 mappings for 16 possible traffic classes).

Next, we map traffic classes to TX queues of the network device. Modern network devices typically implement more than one TX queue for outgoing traffic. How many TX queues are supported by your device, you can find out with the following command:

```
$ ls /sys/class/net/enp2s0f1/queues/
rx-0 rx-1 rx-2 rx-3 rx-4 rx-5 rx-6 rx-7 tx-0 tx-1 tx-2 tx-3 tx-4 tx-5 tx-6 tx-7
```

Here, the network device enp2s0f1 supports 8 TX queues.

The parameter `queues 1@0 1@1` of TAPRIO reads like this: The first entry `1@0` defines the mapping of the first traffic class (TC0) to TX queues, the second entry `1@1` the mapping of the second traffic class (TC1), and so on. Each entry defines a range of queues to which the traffic class should be mapped using the schema queue_count@queue_offset. That is, `1@0` means map (the traffic class) to 1 TX queue starting at queue index 0, i.e., queue range [0,0]. The second class is also mapped to 1 queue at queue index 1 (`1@1`). You can also map one traffic class to several TX queues by increasing the count parameter beyond 1. Make sure that queue ranges do not overlap.

Next, we define the schedule of the TAS implemented by TAPRIO. First of all, we need to define a base time as a reference for the cyclic schedule. Every scheduling cycle starts at base_time + k*cycle_time. The cycle time (duration of the cycle until it repeats) is implicitly defined by the sum of the times (interval durations) of the schedule entries (see below), in our example 800000 ns + 200000 ns = 1000000 ns = 1 ms. The base time is defined in nano seconds according to some clock. The reference clock to be used is defined by parameter clockid. CLOCK_TAI is the International Atomic Time. The advantages of TAI are: TAI is not adjusted by leap seconds in contrast to CLOCK_REALTIME, and TAI refers to a well-defined starting time in contrast to CLOCK_MONOTONIC.

Finally, we need to define the entries of the Gate Control List, i.e., the points in time when gates should open or close (or in other words: the time intervals during which gates are open or closed). For instance, `sched-entry S 01 800000` says that the gate of TC0 (least significant bit in bit vector) opens at the start of the cycle for 800000 ns duration, and all other gates are closed for this interval. Then, 800000 ns after the start of the cycle, the entry `sched-entry S 02 200000` defines that the gate of TC1 (second bit of bit vector) opens for 200000 ns, and all other gates are closed.

# Software TSN Bridge

Now that we know how to use the TAPRIO QDisc, we next set up a software TSN bridge. The software TSN bridge integrates three parts:

* A software bridge taking care of forwarding packets to the right outgoing port.
* Per outgoing switch port (network interface) a TAPRIO QDISC.
* A mapper for mapping PCP values to SKB priorities as used by TAPRIO for classifying packets. 

As an example, we would like to implement the following scenario with two network namespaces hosting applications---network namespaces are typically used by containers, however, we could also use VMs:

![Scenario]({{ site.url }}{{ site.baseurl }}/assets/images/software_tsn_bridge_scenario.png "Scenario"){: .align-center}

If you want to follow this example, you can set up the namespaces as follows:

```
$ sudo ip netns add talker
$ sudo ip netns add listener
```

We set up a software bridge called vbridge and bring it up:

```
$ sudo ip link add name vbridge type bridge
$ sudo ip link set dev vbridge up
```

We assume that each namespace is attached to the virtual bridge with a virtual Ethernet (veth) interface. veth interfaces actually come as pairs of devices, like a virtual cable with two ends. One end (`veth-...-a`) will be attached to the container (namespace), the other end (`veth-...-b`) to the virtual bridge, like you would connect a physical host to a physical bridge. We also need to make sure that veth devices with a TAPRIO QDisc have the required number of TX queues as mentioned above using option `numtxqueues`. In our system, only the virtual TSN bridge uses TAPRIO to schedule packets towards the listener namespace. Therefore, we only set the number of TX queues for the bridge-side veth device `veth-listener-b` towards the listener:

```
$ sudo ip link add veth-talker-a type veth peer name veth-talker-b
$ sudo ip link add veth-listener-a numtxqueues 8 type veth peer name veth-listener-b numtxqueues 8
```

Since TSN requires VLAN tags to carry PCP values, we also create VLAN interfaces (VLAN id 100) for one end of the virtual cable (the end attached to the namespace. All packets coming out of this VLAN device (from the namespace to the bridge) will carry a VLAN tag; for all incoming packets (from the bridge to the container), the VLAN tag is removed. The sending application (called talker in the following) defines SKB priorities as described above using the SO_PRIORITY socket option. This SKB priority is mapped to the PCP value of the VLAN header using the `SO_PRIORITY` option. This ensures that all packets arriving at the bridge have a VLAN tag with defined PCP value. This also applies to packets coming from the physical network outside of the host, i.e., the bridge receives over all attached interfaces VLAN-tagges packets with PCP field:     

```
$ sudo ip link add link veth-talker-a name veth-t-a.100 type vlan id 100
$ sudo ip link add link veth-listener-a name veth-l-a.100 type vlan id 100  
$ sudo ip link set veth-t-a.100 type vlan egress 0:0 1:1
$ sudo ip link set veth-l-a.100 type vlan egress 0:0 1:1
```

We also assign a physical network interfaces (enp2s0f0) to the bridge to connect the bridge to the physical network of the edge server. We put this interface into promiscuous mode, so the virtual bridge will see all incoming packets (similar to physical bridges):

```
$ ip link set dev enp2s0f0 promisc on
```

Then, we assign all network interfaces to the bridge and to the containers, respectively:

```
$ sudo ip link set veth-t-a.100 netns talker
$ sudo ip link set veth-talker-b master vbridge
$ sudo ip link set veth-l-a.100 netns listener
$ sudo ip link set veth-listener-b master vbridge
$ sudo ip link set enp2s0f0 master vbridge
```

We must also bring all interfaces up:

```
$ sudo ip link set veth-talker-a up
$ sudo ip link set veth-talker-b up
$ sudo ip link set veth-listener-a up
$ sudo ip link set veth-listener-b up
$ sudo ip netns exec talker veth-t-a.100 up
$ sudo ip netns exec listener veth-l-a.100 up
```

To perform traffic shaping on egress traffic, we need to assign TAPRIO QDiscs to all network interfaces shaping traffic in egress direction. In our example, we only define a TAPRIO QDisc for the virtual bridge interface towards the listener namespace to demonstrate the idea (in practice, all interfaces attached to the TSN bridge might have a TAPRIO QDisc). SKB priority 0 is mapped to traffic class 0; SKB priority 1 is mapped to traffic class 1 (all other SKB priorities are mapped to traffic class 0). The gate for traffic class 0 is always open (least significant bit always set); the gate for traffic class 1 is 100 ms closed and 50 ms open (cycle time is 150 ms): 

```
$ sudo tc qdisc replace dev veth-listener-b parent root handle 100 taprio \
num_tc 2 \
map 0 1 0 0 0 0 0 0 0 0 0 0 0 0 0 0 \
queues 1@0 1@1 \
base-time 1554445635681310809 \
sched-entry S 01 100000000 sched-entry S 03 50000000 \
clockid CLOCK_TAI
```

Finally, we need to remember that the virtual bridge receives VLAN-tagges packets with PCP values, but TAPRIO needs SKB priorities to classify packets, i.e., we need to perform a mapping from PCP values to SKB priorities. A very elegant method working without removing the VLAN tag is to use a so-called ingress QDisc on all interfaces of the virtual bridge together with the action `skbedit`. This action does exactyl what its name suggests: modify the SKB. In our case, we specify a mapping from PCP value 1 (`vlan_prio 1`) to SKB priority 1 (`priority 1`). For the sake of a short description, we only do this for the interface from namespace `talker`, but you typically would do this on all interfaces of the bridge:

```
$ sudo tc qdisc add dev veth-talker-b ingress
$ sudo tc filter add dev veth-talker-b ingress prio 1 protocol 802.1Q flower vlan_prio 1 action skbedit priority 1
```

# Test

Finally, we can test our software TSN bridge that we have set up above. We start a talker in namespace `talker` sending UDP messages as fast as possible to the listener in namespace `listener` with SKB priority 1, which will be mapped to the PCP 1 by the VLAN device. PCP 1 will be mapped by the ingress QDisc to SKB 1 at the virtual bridge. TAPRIO maps SKB priority 1 to traffic class 1. The gate for traffic class 1 is open for 50 ms and closed for 100 ms. Therefore, we would expect to see this pattern in the traffic arriving at the talker.

We start root shells in the talker and listener namespaces:

```
$ sudo ip netns exec talker /bin/bash
$ sudo ip netns exec listener /bin/bash
```

Then set private IP addresses for talker and listener:

```
(talker) $ ip address add 10.0.1.1/24 dev veth-t-a.100
```

```
(listener) $ ip address add 10.0.1.2/24 dev veth-l-a.100
```

On the listener side, we use netcat to receive and drop all received packets to port 6666:

```
(listener) $ nc -u -l -p 6666 > /dev/null
```

On the talker side, we use a custom C application, which sends UDP packets at a rate of 1000 pkt/s and sets the priority to 1 using the socket option SO_PRIORITY (an alternative application that can also set the required SO_PRIORITY socket option would be socat).

Traffic is captured with TCP dump at the virtual listener interface, i.e., before VLAN tags are removed, so we can also check the correct tagging of packets with VLAN ID 100 and PCP value 1:

```
$ sudo tcpdump -i veth-listener-a -w trace.pcap --time-stamp-precision=nano
```

The following figure shows the arrival times of packets at the listener. We draw a vertical line whenever a packet of stream is received by the listener (the individual lines blend together at higher data rates). 

![Time-shaped traffic]({{ site.url }}{{ site.baseurl }}/assets/images/taprio-traffic.png "Time-shaped traffic"){: .align-center}

As we can see, the packets arrive with the anticipated pattern in bursts of 50 ms length (gate for prio 1 open), so time-aware shaping is effective and working correctly.

Using Wireshark, we can also inspect the packets received at the listener interface `veth-listener-a`, i.e., before VLAN tags are removed:

![Packet inspection with Wireshark]({{ site.url }}{{ site.baseurl }}/assets/images/taprio-wireshark.png "Packet inspection with Wireshark"){: .align-center}

We can see that packets indeed carry a VLAN tag with id 100, and the PCP value is 1, so also the VLAN tagging and PCP mapping works as intended. 
