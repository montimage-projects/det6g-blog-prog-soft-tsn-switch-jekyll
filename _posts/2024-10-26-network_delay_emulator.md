---
title:  "Network Delay Emulator: Emulating the Characteristic 5G/6G Network Delay with Linux"
date:   2024-10-26 00:00:00 +0100
categories: linux tsn 5g 6g networking emulation
tags: linux TSN networking emulation 5G 6G
author:
  name : "Frank Dürr"
  bio : "Senior researcher and lecturer at University of Stuttgart"
  avatar : "/assets/images/photo-frank_duerr.jpg"
  links :
    - label: Homepage
      icon: "fas fa-fw fa-home"
      url: "https://deterministic6g.eu"
    - label: Blog
      icon: "fas fa-fw fa-blog"
      url: "https://deterministic6g.eu"
    - label: "Twitter"
      icon: "fab fa-fw fa-twitter-square"
      url: "https://twitter.com/DETERMINISTIC6G"
    - label: "Github"
      icon: "fab fa-fw fa-github"
      url: "https://github.com/DETERMINISTIC6G"
    - label: "LinkedIn"
      icon: "fab fa-fw fa-linkedin"
      url: "https://www.linkedin.com/company/deterministic6g/"
    - label: "YouTube"
      icon: "fab fa-fw fa-youtube"
      url: "https://www.youtube.com/@DETERMINISTIC6G/videos"
---
In this blog post, we present the DETERMINISTIC6G Network Delay Emulator for Linux.

# tl;dr -- Takeaway Messages

* The performance of networked real-time systems depends on the network delay between distributed components.
* Novel networked real-time systems require wireless communication, using for instance 5G/6G mobile networks.
* The [DETERMNISTC6G Network Delay Emulator](https://github.com/DETERMINISTIC6G/NetworkDelayEmulator) for Linux is an open-source tool to evaluate the performance of networked real-time systems with emulated characteristic network delay, including the possibility to use [open data](https://github.com/DETERMINISTIC6G/deterministic6g_data) from delay measurements in real 5G networks.  
* This blog post presents the DETERMINISTIC6G Network Delay Emulator, its usage, and a showcase (evaluation of an inverted pendulum).

# Motivation 

Networked real-time systems are typically sensitive to network delay, i.e., their performance and safety depends on the delay of communicating messages between distributed system components. Moreover, many novel networked real-time systems include mobile devices that communicate with remote components over a wireless network such as 5G or future 6G mobile networks. The DETERMINISTIC6G project describes a number of such applications in [this document](https://deterministic6g.eu/images/deliverables/DETERMINISTIC6G-D1.1-v1.0.pdf), including:

* Automated guided vehicles moving on a shop floor in a factory and communicting with machines and edge servers in the factory.
* Smart farming where for instance drones monitore the field in front of a harvester to protect animals.
* Exoskeletons assisting workers on a shop floor, which are remotely controlled from an edge cloud server in the factory.
* Extended reality devices like Augmented Reality (AR) headsets displaying remotely rendered images. 

One major challenge comes from the fact that the characteristic network delay of 5G/6G mobile networks is fundamentally different from wired networks, such as a wired Ethernet network. The following figure compares the delay measured for a single TSN Ethernet bridge (left) to the delay of a wireless 5G bridge (right)---open data of these and other delay measurements from the DETERMINISTIC6G project can be found in [this Github repository](https://github.com/DETERMINISTIC6G/deterministic6g_data). Obviously, the delay of the wireless bridge is substantially larger (milli-seconds vs. micro-seconds), stochastic in nature, and heavy-tailed, i.e., the probability of large delay values does not decrease exponentially.

![Port-to-port delay]({{ site.url }}{{ site.baseurl }}/assets/images/port-to-port-delay.png "Port-to-port delay of wired bridge (left) compared to wireless bridge (right)"){: .align-center}

This brings up the crucial question: How would a networked real-time system perform with such a given network delay? For instance, would a networked control system still be stable? What is the Quality of Control (QoC)? Or what Quality of Experience (QoE) would users of a distributed XR system have with such a network delay?  

The answer to these and other questions requires tools to evaluate the performance with characteristic network delay distributions. Network delay emulation is an attractive method to this end, since it allows for testing the real application with an emulated network behaving like a real network---in particular, we want to emulate the characteristic end-to-end network delay of a 5G/6G (or other) network, and see how the real application under test behaves. Testing the real application (if available) is a huge benefit, since building simulation models of the application such as the plant of a control system (e.g., the exoskeleton mentioned above) or even the user in the XR use case mentioned above might be very difficult. Instead, we can use the real application "under test" and expose it to the emulated network delay as shown in the example of a networked control system in the following figure. On the left-hand side and right-hand side, we see real system components, namely the plant of the control system (mobile exoskeleton, drone, automated guided vehicle, etc.), and the controller hosted in an edge cloud environment, respectively. The wireless 5G/6G network is emulated and induces the characteristic network delay of the wireless network.    

![Network emulation]({{ site.url }}{{ site.baseurl }}/assets/images/network_emulation.png "Network emulation"){: .align-center}

The [DETERMINISTIC6G Network Delay Emulator](https://github.com/DETERMINISTIC6G/NetworkDelayEmulator) is an open-source network delay emulator for Linux. In particular, it can emulate network delay from histograms captured through measurements in real networks, such as the openly available [DETERMINISTIC6G delay measurement data](https://github.com/DETERMINISTIC6G/deterministic6g_data).

Next, we will describe the technical details and usage of the Network Delay Emulator in more detail, and also showcase its usage using a textbook example of a networked control system: an inverted pendulum. For more details on how to compile and use the emulator, please visit the [Github page of the DETERMINISTIC6G Network Delay Emulator](https://github.com/DETERMINISTIC6G/NetworkDelayEmulator).

# Architecture of the Network Delay Emulator

The core of the emulator is a Linux Queueing Discipline (QDisc) called sch_delay that can be assigned to network interfaces to add artificial delay to all packets leaving through this network interface. 

The following figure shows the system architecture consisting of two major parts: the QDisc running in the kernel space, and a user-space application providing individual delays for each transmitted packet through a character device. The provided delays are buffered in the QDisc, such that delay values are available immediatelly when new packets arrive. Whenever a packet is to be transmitted through the network interface, the next delay value is dequeued and applied to the packet before passing it on to the network interface (TX queue).

![Network Emulator Architecture]({{ site.url }}{{ site.baseurl }}/assets/images/network-emulator-architecture.png "Network Emulator Architecture"){: .align-center}

Providing delays through a user-space application allows for a flexible and convenient definition of delays without touching any kernel code. The project contains a sample user-space application implemented in Python to define delays as constant values, normal distributions (probability density function), or histograms. This application can be easily extended to calculate other delay distributions.

The QDisc can also be applied to network interfaces that are assigned to a virtual bridge as described below to apply individual delay distributions to packets forwarded through different egress interfaces. This allows for emulating the end-to-end network delay of a whole emulated network with a single Linux machine.

One limitation of this approach based on pre-calculating and buffering delays is that it is restricted to independent and identically distributed (i.i.d.) delays. If the delay of a specific packet depends on the delay of an earlier packet, this cannot be easily modelled by this approach since delays were already calculated and buffered possibly long before the packet to be delayed actually arrives. Also changing the delay distribution at runtime is not easily possible due to the buffering of delays from the old distribution. 

QDiscs are typically configured with the tc (traffic control) command in Linux. Since the sch_delay QDisc requires specific parameters as shown next, a modified version of tc is required (a patch for tc is also included in the GitHub repository of the Network Delay Emulator). The following parameters are available to configure the QDisc:

| Option | Type | Default | Explanation |
|--------|------|---------| ----------- |
| limit  | int  | 1000    | The size of the internal queue for buffering delayed packets. If this queue overflows, packets will get dropped. For instance, if packets are delayed by a constant value of 10 ms and arrive at a rate of 1000 pkt/s, then a queue of at least 1000 pkt/s * 10e-3 s = 10 pkts would be required. A warning will be posted to the kernel log if messages are dropped. |
| reorder| bool | true    | Whether packet reordering is allowed to closely follow the given delay values, or keep packet order as received. If packet reordering is allowed, a packet with a smaller random delay might overtake an earlier packet with a larger random delay in the QDisc. If packet re-ordering is not allowed, additional delay might be added to the given delay values to avoid packet re-ordering. |

# Emulating End-to-End Network Delay for Multiple End-to-End Paths

Assume you want to emulate the individual end-to-end network delay in different directions (upstream and downstream to/from hosts) and/or between different pairs of hosts. To this end, you can use a virtual bridge on a central emulation node and assign individual delays to each outgoing (downstream) port.

The following figure shows a sample topology with two hosts Host1 and Host2, respectively. The end-to-end delay of packets from Host1 to Host2 shall be different from the end-to-end delay from Host2 to Host1. The machine in the middle is the host implementing network emulation to introduce the artificial delay between Host1 and Host2. A Linux virtual bridge is used to forward packets from eth0 to eth1 and vice versa. The two depicted QDiscs add the delay to egress packets individually in each direction. 

![Sample topology]({{ site.url }}{{ site.baseurl }}/assets/images/network_emulation_topology.png "Sample topology"){: .align-center}

To implement this scenario, we first create a virtual bridge on HEmu and assign the two physical network interfaces eth0 and eth1 to the virtual bridge. We also bring the interfaces up in case they were previously down:

```console
$ sudo ip link add name vbridge type bridge
$ sudo ip link set dev vbridge up
$ sudo ip link set eth0 master vbridge
$ sudo ip link set eth1 master vbridge
$ sudo ip link set eth0 up
$ sudo ip link set eth1 up
```

Next, we add the delay QDiscs to eth0 and eth1 on Hemu using the tc command:

```console
$ cd ~/NetworkDelayEmulator/tc
$ sudo ./iproute2/tc/tc  qdisc add dev eth0 root handle 1:0 delay reorder True limit 1000
$ sudo ./iproute2/tc/tc  qdisc add dev eth1 root handle 1:0 delay reorder True limit 1000
```

Finally, we start two instances of the user-space application, one providing delays for messages leaving through port eth0 (character device `/dev/sch_delay/eth0-1_0`) to emulate the delay from Host2 towards Host1, and one for port eth1 (character device `/dev/sch_delay/eth1-1_0`) to emulate the delay from Host1 to Host2.

```console
$ cd ~/NetworkDelayEmulator/userspace_delay
$ sudo python3 userspace_delay.py /dev/sch_delay/eth0-1_0
$ sudo python3 userspace_delay.py /dev/sch_delay/eth1-1_0
```

The sample user-space application provided with the emulator is an interactive Python application presenting the user with different options to specify delays---of course, you could also use your own custom application to create delay values porgrammatically and send them to the QDisc. In particular, the provided user-space application can load delay histograms created from measurements. The DETERMINSTIC6G project provides different delay data sets measured in real 5G networks in [this GitHub repository](https://github.com/DETERMINISTIC6G/deterministic6g_data). How to integrate these measurements with the emulator is described in detail in the README file of the emulator.  

# Per-Traffic-Class Delay QDiscs

QDiscs are a sophisticated concept in Linux. We can exploit this concept to build a hierarchy of QDiscs to assign different delays to packets of different traffic classes sent through the same network interface.

For instance, the QDisc hierarchy could look as follows:

```
            ETS
	   (1:0)
	  /     \
	 /       \
       ETS       ETS
       1:1       1:2
        |         |
	|         |
      Delay     Delay
     (10:0)     (20:0)
```

At the top of the hierarchy, we place an ETS (Enhanced Transmission Selection) QDisc. ETS is a so-called classful QDisc with sub-classes to which other QDiscs can be assigned. The traffic of these sub-classes is scheduled by ETS either using deficit round-robbin (weighted fair bandwidth sharing) or priority queuing. We use fair bandwidth sharing since we do not want to penalize any of the traffic classes (other than applying some delay to them). The ETS QDisc with two sub-classes is set up as follows on a network interface, say `eth0`:

```
$ sudo tc qdisc add dev ens1f3 root handle 1: ets bands 2 quanta 1000 1000
```

Each band receives the same "quanta" for fair sharing (same bandwidth for all sub-classes), and is implicitly associated with a sub-class. The first sub-class gets the handle 1:1 and the second 1:2 (parent:bandnumber).

Next, we add two delay QDiscs to sub-class 1:1 and 1:2:

```
$ sudo tc qdisc add dev eth0 parent 1:1 handle 10: delay reorder True limit 1000
$ sudo tc qdisc add dev eth0 parent 1:2 handle 20: delay reorder True limit 1000
```

The first delay QDisc gets the handle 10: and the second the second the handle 20:.

Finally, we set up filters to classify egress traffic from eth0 as follows:

```
$ sudo tc filter add dev eth0 protocol ip parent 1: prio 1 u32 match ip dst 192.168.1.1/24 flowid 1:1
$ sudo tc filter add dev eth0 protocol ip parent 1: prio 2 matchall flowid 1:2

```

The first filter has the highest priority (prio 1). The second filter has a lower priority (prio 2) and, therefore, is only effective if the first filter does not match.

The first filter uses the versatile u32 filter, which matches on bits of the packet header. `ip dst ...` is just a shorthand for matching on the bits of the IP destination address of IPv4 packets. Such packets are assigned to traffic class 1:1 and, therefore, will be delayed by the delay QDisc 10:.

The second filter catches all other packets (matchall) and assigns them to the second traffic class, which is delayed by the second delay QDisc 20:0.

For other classful QDiscs and filters, please check the man page of tc.

# Evaluation of Delay Emulation Accuracy

To give an impression on the accuracy to be expected with the NetworkDelayEmulator, we performed measurements with the following virtual bridge setup. We used network taps in fiber optic cables (marked with `x`) and an FPGA network measurement card from Napatech (NT40E3-4-PTP) to capture the traffic from H1 (sender) to H2 (receiver) with nano-second precision.

```
       pcap (sender)
         ^
         |
 ----    |     -------------------------------------          ----
|    |   |    |              ---------              |        |    |
|    |---x--->|------------>|         |<------------|<-------|    |
| H1 |        | eth0        | vBridge |        eth1 |        | H2 |
|    |<-------|<------------|         |------QDisc->|---x--->|    |
|    |        |              ---------              |   |    |    |
|    |        |                Hemu                 |   |    |    |
 ----          -------------------------------------    |     ----
                                                        |
                                                        v
                                                       pcap (receiver)
```

The sender app on H1 sends minimum-size (64B) UDP packets at a rate of 100 pkt/s and at a speed of 10 Gbps to the receiver app on H2. 

The QDisc is configure with a normal distribution with mean = 10 ms and stddev = 1 ms.

As baseline, we also capture a trace with zero delay emulation (w/o QDisc on eth1). 

Traces were captured for about 10 min.

The specs of the Hemu host are:

* Intel(R) Xeon(R) CPU E5-1650 v3 @ 3.50GHz
* 16 GB RAM

The following figure shows the histogram of the measured (emulated) end-to-end delay overlayed with the input histogram. We see that the emulated delay closely follows the input with a small offset (see below).

![Network emulation accuray]({{ site.url }}{{ site.baseurl }}/assets/images/emulation-accuracy.png "Network emulation accuray"){: .align-center}

The following figures show the histograms of the actual delay between the measurement points. Theoretically, we would need to subtract:

* One transmission delay (64*8 bit / 10 Gbps = 51.2 ns) since Hemu will only start processing the packet when it has been fully received, and the measurement card takes the timestamp at the header.
* The propagation delay for about 6 m fiber cable (about 6 m / (2/3*3e8 m/s) = 30 ns) connecting the tap to the measurement card.

However, since this delay is only in the range of microseconds or below, we report values as measured. 

![End-to-end delay normal distribution]({{ site.url }}{{ site.baseurl }}/assets/images/e2e_delay_normal_distribution.png "End-to-end delay with normal distribution"){: .align-center}
![End-to-end delay null distribution]({{ site.url }}{{ site.baseurl }}/assets/images/e2e_delay_null_delay.png "End-to-end delay null distribution"){: .align-center}

For the normal distribution, the following values were measured:

* mean = 0.010126313739089609 s
* stddev = 0.0009962691742813061 s
* 99 % confidence interval of the mean = [0.010115940597227065 s, 0.010136686880952152 s]
* min = 0.005947589874267578 s
* max = 0.014116764068603516 s

Without delay emulation, the delay was:

* mean = 8.166161013459658e-05 s
* 99 % confidence interval of the mean = [8.159266669171887e-05 s, 8.173055357747428e-05 s]
* min = 1.0251998901367188e-05 s
* max = 9.942054748535156e-05 s

We see that the delay added without any emulated delay is around 81 us. This could be considered as an offset when creating delay distributions.

# Evaluation of a Networked Control System with Emulated Network Delay

As motivated in the beginning, network emulation is a great method to evaluate the influence of network delay onto already existing, real applications. To showcase this ability, we use a textbook example in the following: an inverted pendulum (networked control systems), which is remotely controlled from an edge cloud server.

The following figure shows the setup of the evaluation. As inverted pendulum, we use a software implementation (simulation) of the pendulum available [here on GitHub](https://github.com/duerrfk/Inverted-Pendulum). But of course, you could also use a real pendulum, as long as it can communicate through the emulator host with a remote controller on another machine in the network.

![Emulator with inverted pendulum]({{ site.url }}{{ site.baseurl }}/assets/images/network-emulator-pendulum.png "Emulator with inverted pendulum"){: .align-center}

We run the experiment once without adding extra emulated network delay (only delay of a wired 10 Gbps Ethernet network and the software bridge running on the emulator host), and once with the characteristic emulated network delay from a 5G system as made openly available in [this GitHub repository](https://github.com/DETERMINISTIC6G/deterministic6g_data) by the DETERMINISTIC6G project. The pendulum is initialized with a 5 degree angle (deviation from the zero degree setpoint).

The following figure shows the angle of the pendulum over time, which corresponds to the error of the controlled system (deviation from zero degree upright position of the pendulum). We can see that with the emulated delay of the 5G system, it takes significantly longer to bring the pendulum to the zero degree setpoint, although both pendulums did not tip over. 

![Inverted pendulum evaluation]({{ site.url }}{{ site.baseurl }}/assets/images/inverted-pendulum-evaluation.png "Inverted pendulum evaluation"){: .align-center}

Of course, a more detailed evaluation and rigorous analysis of the stability and performance of the system would be required for any real system. However, this demonstrations might suffice to show the idea of how to use the network delay emulator.

# Acknowledgments

NetworkDelayEmulator was originally developed by Lorenz Grohmann in his [Bachelor's thesis](https://elib.uni-stuttgart.de/handle/11682/14240) at University of Stuttgart in the context of the DETERMINISTIC6G project, which has received funding from the European Union's Horizon Europe research and innovation programme under grant agreement No. 101096504.
