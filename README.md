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

##4. Create Additional RX/TX Queues
In the following step, we will add a new queue while operating in TAP mode. However, before proceeding, it is recommended to retrieve relevant configuration details using the show config fwd command.
##5. Create Flow Filtering Rule in testpmd
To direct specific types of traffic to designated queues, you can create a flow filtering rule in testpmd using the following command.
`flow create 0 ingress pattern eth / ipv4 / udp / end actions queue index 0 / end`
##5. Create Flow Filtering Rule in testpmd
To direct specific types of traffic to designated queues, you can create a flow filtering rule in testpmd using the following command.
`flow create 0 ingress pattern eth / ipv4 / udp / end actions queue index 0 / end`
##6. Install and Run tcpreplay
install [tcpreplay](https://github.com/appneta/tcpreplay/releases/tag/v4.5.1) from main source.

