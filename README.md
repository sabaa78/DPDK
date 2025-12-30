# DPDK
Deep_Linux
# Deep_Linux — DPDK Flow Rule CPU Cost (TAP + testpmd + LTTng)

This project investigates the CPU processing cost imposed by configuring **DPDK Flow Rules** when using the **TAP PMD/driver** with **DPDK testpmd**, and aims to identify *where the cost comes from*.

## Goals

We want to quantify and explain the overhead introduced by flow-based filtering and steering, focusing on:

- **UDP / TCP filtering**
- **Directing packets to a specific RX queue**
- **How expensive header parsing is**
- **Why LTTng does not capture some behaviors / time spent**
- **End-to-end path:** `tcpreplay → TAP → DPDK testpmd → (Flow Rule + LTTng tracing)`

## Steps
We can see steps of project here:

- **We create synthetic traffic (pcap)**
- **We send it through TAP**
- **DPDK processes it**
- **LTTng records every function called**
- **TraceCompass analyzes where the CPU time goes**

# Installing and Building DPDK with Function Tracing Support

This document describes how to build **DPDK with user-space function tracing enabled** in order to analyze CPU processing cost using **LTTng**, particularly when using **testpmd** with the **TAP Poll Mode Driver (PMD)**.

---

## 1. Build DPDK with Function Tracing Enabled

### 1.1 Download the latest DPDK release

Retrieve the latest stable release from the official DPDK website:

- https://www.dpdk.org/download/

---

### 1.2 Extract the archive

```bash
tar xJf dpdk-<version>.tar.xz
cd dpdk-<version>
```
### 1.3 Configure the build environment (Meson)
```bash
meson setup build \
  -Dexamples=all \
  -Dlibdir=lib \
  -Denable_trace_fp=true \
  -Dc_args="-finstrument-functions"
```

Explanation

-Denable_trace_fp=true
- **Enables DPDK trace fast-path instrumentation.**
- **-Dc_args="-finstrument-functions"**
- **Instruments function entry and exit points at compile time.**
- **This flag is mandatory for LTTng user-space function tracing:**
- **Without it, LTTng may produce empty or incomplete traces**
- **Required to observe detailed DPDK packet-processing paths**


### 1.4 Build and Install Using Ninja
```bash
cd build
ninja
meson install
ldconfig
```
## 2. Configure hugepages and mount 1GB pagesize
```bash
echo 1024 > /sys/kernel/mm/hugepages/hugepages-2048kB/nr_hugepages
mkdir /mnt/huge
mount -t hugetlbfs pagesize=1GB /mnt/huge
```
Alternatively, the following method can be used:
```bash
sudo sysctl -w vm.nr_hugepages=1024
mount -t hugetlbfs none /dev/hugepages
```
## 3. Create two TAP interfaces for DPDK's TAP Poll Mode Driver (PMD)
Within the dpdk-/build directory, execute the testpmd application using the following command:
```bash
sudo LD_PRELOAD=/usr/lib/x86_64-linux-gnu/liblttng-ust-cyg-profile.so.1 ./app/dpdk-testpmd -l 0-1 -n 2   --vdev=net_tap0,iface=tap0   --vdev=net_tap1,iface=tap1   --   -i
```
### Functionality:
- **Creates net_tap0 and net_tap1 virtual devices**
- **Assigns 2 CPU cores (-l 0-1)**
- **Starts in interactive mode (-i)**

## 4. Create Additional RX/TX Queues

In the following step, we will add a new queue while operating in TAP mode. However, before proceeding, it is recommended to retrieve relevant configuration details using the show config fwd command.

## 5. Create Flow Filtering Rule in testpmd
To direct specific types of traffic to designated queues, you can create a flow filtering rule in testpmd using the following command.
`flow create 0 ingress pattern eth / ipv4 / udp / end actions queue index 0 / end`

## 6. Install and Run tcpreplay
install [tcpreplay](https://github.com/appneta/tcpreplay/releases/tag/v4.5.1) from main source.
#### Note:
remind to run tcpreplay with these flag:
```bash
./configure --disable-tuntap
make
sudo make install`
```
Next, in the `testpmd` terminal window, enter the command `start` , followed by `show port stats all` to observe the packet flow on TAP interface 0.

# Tracing CPU and Packets with LTTNG
In order to Automate the LTTng capture, create a shell script to configure the LTTng session. The script initializes the session, adds the necessary context fields, starts tracing, sleeps for a specified duration, and then stops and destroys the session.
```bash
touch script.sh
chmod +x script.sh
nano script.sh
```
Paste the following commands into the file:
```bash
#!/bin/bash
lttng create libpcap
lttng enable-channel --userspace --num-subbuf=4 --subbuf-size=40M channel0
#lttng enable-channel --userspace channel0
lttng enable-event --channel channel0 --userspace --all
lttng add-context --channel channel0 --userspace --type=vpid --type=vtid --type=procname
lttng start
sleep 10
lttng stop
lttng destroy
```
### Explanation
This script creates an LTTng tracing session named `libpcap` , configures a userspace tracing channel `(channel0)` with a fixed ring-buffer setup (4 sub-buffers, 40 MB each), and then enables all userspace tracepoints/events on that channel. It also enriches every recorded event with extra context fields—virtual process ID (vpid), virtual thread ID (vtid), and the process name (procname)—to make analysis easier. After starting the trace, it records for 10 seconds, then stops tracing and destroys the session (cleaning up the session and its configuration).

<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/cf3a46d5-ad63-4457-b973-df068792407c" /> <img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/10840826-f0eb-4f49-9be0-9ca1c050ed1d" />


As can be seen:

In the output of the first image, which is related to the state in which the rule is set, we see a total of 4,385,297 events recorded in 1 second
However, in the output of the second image, which is related to the state without the rule, we see 4,554,239 events in the same time range. 
This output shows that in the no-rule mode we recorded more events.
In the case with rule we have more valleys:
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/72cd6bb7-5d1f-4671-a8ed-910da4306102" />
## `pmd_rx_burst`
After the comparison made in the two below modes, we arrive at this function `pmd_rx_burst` which has a time difference of 1 second between the two modes with and without rules in the entire process of 100 times.
<img width="1850" height="965" alt="image" src="https://github.com/user-attachments/assets/0800e298-3e69-4cd5-9b1f-110bfe29644f" />
<img width="1850" height="965" alt="image" src="https://github.com/user-attachments/assets/cd6cfab2-3a97-4a83-b407-f98ae7f99371" />

### checking Event Desnsity 
## without rule:
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/7c65a252-f1ec-48b9-9204-9e94c15a5d21" />

## with rule:
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/11b0aa5a-098a-4332-9ff1-ac2bac652b96" />




