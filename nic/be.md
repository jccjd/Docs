## NIC Performace 

performance

- performance metrics

- Factors that impact performance

- Broadcom NIC Performance Tuning Guide and Reports

- Performance Tuning Commonly Used Commands

- iperf/netperf, IP forwarding, DPDK, RoCE perftest

- Debugging Performance issues

  

### Throughput 

-  The amount of data that is sent/received in one second
-  Mpps(fps) or Gbps



|                     | bits on the wire                      | bits               | overhead |
| ------------------- | ------------------------------------- | ------------------ | -------- |
| one 64B frame       | (7 + 1 12 ) \* 8 + 64B \* 8 = 672 bit | 64B\*8=512bit      | 23.8%    |
| iperf TCP MTU=1500B | 20B\*8 +1518B\*8=12305bit             | 1460B\*8=11680 bit | 5%       |
|                     |                                       |                    |          |

### Latency

- The time required to send/receive a packet
- One-way ore round-trip (in microseconds or transactions/s)

## Factors that impact NIC performance

| Factors          | Description                                                  |
| ---------------- | ------------------------------------------------------------ |
| CPU              | - Frequency <br />- Architecture: IPC (instructions per cycle)<br />-Power mode: Use performance mode, disable processor C states<br />- SMP: Symmetric Multi-Processing<br />- SMT: Use physical cores before using hyper-threaded cores<br />-NUMA:Across NUMA can result in performance degradation |
| Cache and Memory | Memory bandwidth=Memory frequency(e.g 3200MT/S)\*64bit\*channel_number |
| PCIe             | PCIe bandwidth= bitrate_per_lane(e.g Gen4 16GT/s)\* lane_number (e.g 16 lanes) |
| OS/Kernel        | - selinux, firewalld/iptables<br />- irqbalance<br />- Iommu<br />- kernel tcp/ip paramentes, AMD Rome/Milan nohz=off |
|                  |                                                              |
|                  |                                                              |



### Single CPU core: How does Linux receive a network packet



Tools for determing CPU utilization

- top
  - hi time spent servicing hardware interrupts
  - si: time spent servicing software interrupts
  - us: time runing user processes
  - sy: time ruing kernel processes
  - id: time spent in the kernel idle handler



## NAPI and Interrupt Coalescing

- NAPI(New API)
  - An interrupt-driven mode by default and switches to polling mode when the flow of incoming packets exceeds a certain threshold
  - Implemented in bnxt_en ethernet driver
- Interrupt Coalescing 
  - A technique in which events which would normally trigger a hardware interrupt are held back, either until a certain amount of work is pending, or a timeout timer triggers 
  - Lowers CPU utilization, increases throughput,but might increases latency
  - Lowers CPU utilization, increases throughput, but might increases latency
  - Changes interrupt coalescing setting by "ethtoo -C"
    - rx-usescs:how many microseconds after at least 1 packet is received before generating an interrupt 
    - rx-frames:how many packets are received before generating an interrupt 
    - adaptive-rx: improve latency under low packet rates and improve throughput under high packer rates 



## NIC Ring Size

- Tx/RX Ring Size

  - TX/RX ring size is buffer descriptor number of a TX/RX ring

  - NIC HW and driver use descriptor rings for sending and receiving packets. Driver is producer, and NIC HW is consumer

  - Larger ring size provides an improved resiliency against the packet loss ,but increases the cache/memory footprint resulting in a performance penalty

  - Change ring size by "ethtool -G"

    - ethtool -G <devname> tx 20487rx 2047

      

## Maximum Transmission Unit

- Larger MTU is associated with fewer packets to be rocessed  and reduced overhead, resulting in much better throughput
  -  Causing delays to subsequent packets, and is problematcin the presence of  communications errors
  -  Not commonly used in read deployment
- Change MTU size by "ifconfig mtu"
  - ifconfig <devname> mtu 15000 # Standard Ethernet Frame Size 1400
  - ifconfig <devname> mtu 9000 # Jumbo Frame size up to 9000



## GRO(Generic Receive Offloading)

- GRO coalesces serveral receive packets from a stream into one large packet. This allows only a single packet to be processed and reduces the overhead, resulting in better performance
- Broadcom NICs support GRO in HW
  - Be able to aggregate SW GRO which is not as good as HW GRO
  - Our competitors use SW which is not as good as HW GRO
- In case the packets could not be aggregated, GRO does not help and might bring performance penalty
- Enable/disable GRO by "ethtool -K"
  - ethtool -K <devname> gro on 
  - ethtool -K <devname> gro on rx-gro-hw on # latest kernel allows "gro on rx-gro-hw off"



## PCIe MRRS and Relaxed Ordering

- Maximum Read Request Size
  - The maximum size of a PCIe memory read request
  - MPS (Maximum Pavload Size) vs MRRS
    - MPS is like the MTU of PCIe. Typically MPS value should follow server's MPS setting
  - Larger MRRS improvers PCIe RX bandwidth in large TLP package cases 
  - Change Broadcom NIC MRRS
    - setpci -s <bus:dev:fun> b4.w=5xxx # 0->128B 1->256B, 2->512B, 3->1024B,4->2048B,5->4096B
    - lspci -s <bus:dev:fun> -vvv | grep MaxReadReq # query current MRRS value
- Relaxed Ordering
  - Allows PCIe packets to be retired out of order when possible.
  - Enable relaxed ordering maintains data consistency and improvers performance in AMD Rome/Milan platform. Much less commonly required on other platfomrs.
  - Enable Broadcom NIC relaxed ordering by bnxtnvm tool
    - bnxtnvm -dev=<devname> setoption=pcie_relaxed_ordering #1



## Multiple CPU Cores: Multiple Rings and RSS

RSS(Receive Side Scaling)

- Provides a good mechanism for RXload distribution as it hashes different streams to separate RX rings to spread the load evenly
- Change ring number by "ethtool -L"
  - ethtool -L <devname> combined 16
  - ethtool -L <devname> combined 0 tx 8 rx 8
- Check packets are balanced to rings
  - watch -n 1 -d "ethtool -S <devname> | grep rx_ucast_packets"







## IRQ Affinity

- IRQ affinity
  - The binding of each ring interrupt to one or multiple CPU cores
  - The distribution of the IRQs(rings) across different CPU cores parallelly results in improved performance
- Linux irqbalance service
  - Linux daemon that balances interupts across all CPU cores automatically
  - Disable  irqbalance to prevent unexpected IRQ affinity
    - systemctl stop irqbalance
- Set IRQ affinity manually by /proc/irq/[interrupt number]/smp_affinity_list
  - an example script to set ring number and IRQ affinity manually
- Check interrupts are balanced to CPU cores
  - watch -n 1 -d "cat /proc/interupts | grep <devname>"



## Run Multiple Instances of Application and Application Affinity

- Run multiple instance of an application
  - Processes are balanced to different CPU cores by kernel schedule algorithm automatically
- Linux application affinity manually by taskset
  - The binding a process to one or multiple CPU cores manually
  - taskset -c 0,2,4,6 iperf3 -s 
  - if using netperf ther is a -T option to handle both server and client application affinity
- Check load are balance to CPU cores
  - "mpstat -P All 1"  or top

## RFS(Receive Flow Steering) and aRFS(Accelerated RFS)

- RSS does not consider application locality. For example, a stream could hash to ring 0 which is being processed on core 0 but the application consuming that data might be running on core 5.This does not benefit from any cache locality

- RFS overcomes this shortcoming by steering the packets the CPU core where the application thread is running and thus increasing the data cache hit rate

- Broadcom NICs support this steering in HW. Configure aRFS

  - ethtool -K <devname> ntuple on 
  - echo 32768 > /proc/sys/net/core/rps_sock_flow_entries
  - echo [rps_flow_value] > /sys/class/net/<devname>/queues/rx-$i/rps_flow_cnt
    - #rps_flow_val = 32768/number of rings

- ethtool -n <devname>

  图

## Broadcom NIC Performance Tuning Guide and Reports



## Commonly Used Commands: System Information

| Command           | Examle                                                       |
| ----------------- | ------------------------------------------------------------ |
| uname             | uname -a                                                     |
| lscpu             | lscpu                                                        |
| cat local_cpulist | cat /sys/class/net/<devname>/device/local_cpulist            |
| lstopo            | lstopo                                                       |
| numactl           | numactl -H                                                   |
| lspci             | lspci -s <bus:dev:fun> -vv\| grep LnkSta<br />lspci -s <bus:dev:fun> -vv \| grep MaxReadReq |
| setpci            | setpci -s <bus:dev:fun:> b4.w=<value>                        |
| kernel config     | cat /proc/cmdline                                            |



## Commonly Used Commands:CPU Status

| Command           | Example                                                      |
| ----------------- | ------------------------------------------------------------ |
| cat /proc/cpuinfo | cat /proc/cpuinfo \| grep -i mhz                             |
| turbostat         | turbostat                                                    |
| pcm               | pcm                                                          |
| top               | top                                                          |
| mpstat            | mpstat -P ALL 1 \| grep -v 100.00$                           |
| perf              | perf top -e cpu-clock -g -d 10                               |
| tuned-adm         | tuned-adm profile [profile_name]                             |
| cpupower          | cpupower frequency-set-g [mode]<br />cpupower idle-set -D 10 |

## Commonly Used Commands: Packets and Interrupts

| Command              | Example                                                      |
| -------------------- | ------------------------------------------------------------ |
| sar                  | sar -n DEV 1 100                                             |
| tcpdump              | tcpdump -i <devname>                                         |
| watch                | watch -n1 -d "ethtool -S <devname> \| grep rx_ucast_packets"<br />watch -n1 -d "cat /proc/interrupts \| grep <devname>" |
| systemctl            | systemctl stop irqbalance                                    |
| cat /proc/interrupts | cat /proc/interrupts \| grep [interface] \| awk -F ":"  '{print $1}'  '' |
| set irq affinity     | echo [cpu_core_mask] > /proc/irq/[interrupt number ]/smp_affinity_list |
| taskset              | taskset -c [cpu_core list] application                       |

## Commonly Used Commands: ethtool

| Command      | Example                                                      |
| ------------ | ------------------------------------------------------------ |
| ethtool  -i  | ethtool -i <devname>                                         |
| ethtool -S   | ethtool -S <devname> \| grep discard                         |
| ethtool -I/L | ethtool -L <devname> combined 16                             |
| ethtool -k/K | ethtool -K <devname> ntuple on<br />ethtool -K <devname> gro on |
| ethtool -c/C | ethtool -C <devname> adaptive-rx on                          |
| ethtool -a/A | ethtool -A <devname> tx off rx off                           |

## Commonly Used Commands:bnxtnvm

- bnxtnvm
  - bnxtnvm is Broadcom NIC firmware update and configuration tool 
  - bnxtnvm -dev=<devname> setoption=pcie_relaxed_ordering enable
  - bnxtnvm -dev=<devname> coredump # retrieve coredump for debugging

### iperf/netperf

- netperf TCP througput test example
  - netserver -p 13000 &
  - netserver -p 13001 &
  - netperf -H 192.168.1.1 -l 60 -t TCP_STREAM -T 0,0 13000,13000 -P 0 -- -N -m=64k -P 36450,35450 &
  - netperf -H 192.168.1.1 -l 60 -t TCP_STREAM -T 1,1 -p 13001, 13001 -P 0 -- -N -n -m=64k -P 36451,35451 &



### netperf tuning

 - BIOS and OS tuning
 - NIC tuning
   - GRO is enableed by default
   - Set IRQ affinity
   - Run multiple instance applications and set application affinity
 - iperf/netperf expected result
   - Line rate for bidirectional TCP throughput
   - Refer to BCM5741X/BCM575XX Performance Indicators    for more details



### IP Forwarding

​	

 - IP Forwarding tuning
   - BIOS and OS tuning
   - NIC tuning
     - ethtool -L <devname> combined 24
     - Set IRQ affinity
     - ethtool -A <devname> tx off rx off
     - ethtool -K <devname> lro off gro off
     - ethtool -C <devname> rx-usecs 512 tx-usecs 512 rx-frames 512 tx-frames 512

### IP Forwarding expected result

- 0.8 ~ 1 mpps per CPU core

- ~15 mpps in 16 CPU cores
- Line rate for 1518B in 16 CPU cores
- Refer to BCM5741X/BCM575XX Performance Indicators  for more details

### DPDK

- testpmd or I3fwd
- DPDK tuning
  - BIOS and OS tuning
  - NIC tuning
    - BCM575XX default settings are good enough for typical DPDK test
- DPDK testpmd io forward expected result
  - ~68 mpps per CPU core in Intel Xeon Gold 6154 CPU at 3.00GHz
  - Refer to http://fast.dpdk.org/doc/perf/DPDK_20_11_Broadcom_NIC_performance_report.pdf for more detail



### RoCE perftest



- perftest

```
- ib_write_bw -x 3 -d <roce_devname> --run_infinitely --report_gbits
- ib_write_bw -x 3 -d <roce_devname> --run_infinitely --report_gbits <server_ip>
- ib_write_lat -x 3 -d <roce_devname>
- ib_write_lat -x 3 -d <roce_devname> <server_ip>
```

- perftest tuning

```
- BIOS and OS tuning
_ NIC tuning
	-BCM575XX default settings are good enough for typical perftes
```



- perftest expected result

```
- Line rate for bi-direction write bandwidth
- ~2us for ib_write typical latency
- Refer to BCM5741X/BCM575XX Performance Indicators for more details

```



### Debugging Performance Issues

- Have BIOS and OS been tuned
- What is throughput number in the single CPU core case
  - Identify the performance bottleneck. Who takes the most of CPU cycles 
    - top, perf
- Why was throughput not improved in multiple CPU cores case
  - Does RSS work and the streams are loaded to multiple rings evenly
    - watch --n 1 -d "ethtool -S <devname> | grep rx_ucase_packets"
  - Does the IRQ affinity work and  workload evenly balanced to CPU cores
    - cat /proc/interrupts, mpstat
  - Check NUMA settings
  - Is it caused by server resource confilcted
  - 













