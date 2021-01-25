# First time hands-on with DPDK sample applications
## Introduction
[To-Do]
## Steps
### 1. Prepare the system
[To-Do]
#### 1.1 Set hugepages
[To-Do]
#### 1.2 Load PMD driver
[To-Do]
#### 1.3 Install dependencies
[To-Do]
```bash
#in a fedora/ubuntu system
apt install meson ninja
```
### 2. Download the dpdk source code
```bash
git clone https://github.com/DPDK/dpdk.git
```
### 3. Compile the source and install globally
```bash
cd dpdk
# meson will configure the project, not build it
meson build #build here is the directory were the build will be generated, not an instruction
cd build
ninja #ninja will compile and produce the object code
#from now on must be run as root
sudo ninja install #this copies the built objects to their final system-wide destination
sudo ldconfig  #tells the dynamic loader to update its cache to include the new objects
```
At this point we have built the dpdk libraries from the source code, but no application has been compiled and cannot yet be executed.

### 4. Compile the desired sample applications
```bash
cd build
meson configure -Dexamples=helloworld,l2fwd #this again configures these 2 projects to be built
ninja #this will compile the configured applications
# you can now check that the objects are under ./examples/helloworld and ./examples/l2fwd
```

### 5. Execute the helloworld application
  In the `dpdk/usertools/` folder we can find several useful scripts. `./cpu_layout.py` prints the CPU layout information. For example:
```
ddeandres@ubuntu:~/dpdk/usertools$ python3 cpu_layout.py
======================================================================
Core and Socket Information (as reported by /sys/devices/system/cpu)
======================================================================
cores =  [0, 1, 2, 3, 4, 5, 6, 8, 9, 10, 11, 12, 13, 14]
sockets =  [0, 1]

        Socket 0        Socket 1
        --------        --------
Core 0  [0, 28]         [14, 42]
Core 1  [1, 29]         [15, 43]
Core 2  [2, 30]         [16, 44]
Core 3  [3, 31]         [17, 45]
Core 4  [4, 32]         [18, 46]
Core 5  [5, 33]         [19, 47]
Core 6  [6, 34]         [20, 48]
Core 8  [7, 35]         [21, 49]
Core 9  [8, 36]         [22, 50]
Core 10 [9, 37]         [23, 51]
Core 11 [10, 38]        [24, 52]
Core 12 [11, 39]        [25, 53]
Core 13 [12, 40]        [26, 54]
Core 14 [13, 41]        [27, 55]
```
We will use the information in this table for the helloworld application as when we execute it, it will print a hello message from the core which we provide the program as argument. For exaple, to say hello from the cpu 14 we will execute:
```bash
sudo ./dpdk-helloworld -l 14
```
which will produce:
```
EAL: Detected 56 lcore(s)
EAL: Detected 2 NUMA nodes
EAL: Multi-process socket /var/run/dpdk/rte/mp_socket
[... output trimmed...]
hello from core 14
```
We can also provide a selection of cpus:
```bash
sudo ./dpdk-helloworld -l 14,16-18
```
which will produce:
```
EAL: Detected 56 lcore(s)
EAL: Detected 2 NUMA nodes
EAL: Multi-process socket /var/run/dpdk/rte/mp_socket
[... output trimmed...]
hello from core 16
hello from core 17
hello from core 18
hello from core 14
```
### 6. Execute the l2fwd application
This application uses 2 ports and copies any packet coming through the first port to the second port. More information and explanation about its implementation can be found in the  [l2fwd official documentation](https://doc.dpdk.org/guides/sample_app_ug/skeleton.html).

Without entering into much details, dpdk requires that the physical port devices are managed using special drivers (Yes, this is the secret sauce which allows dpdk to have such good performance). This means that we must bind them to this driver. Again there is a handy script in the `dpdk/usertools/` called `dpdk-devbind.py`.
Let's first show the current device status:
```bash
python3 dpdk-devbind.py -s
```
Which produces the following output:
```
Network devices using DPDK-compatible driver
============================================

Network devices using kernel driver
===================================
0000:01:00.0 'I350 Gigabit Network Connection 1521' if=ens9f0 drv=igb unused=vfio-pci *Active*
0000:01:00.1 'I350 Gigabit Network Connection 1521' if=ens9f1 drv=igb unused=vfio-pci
0000:01:00.2 'I350 Gigabit Network Connection 1521' if=ens9f2 drv=igb unused=vfio-pci
0000:01:00.3 'I350 Gigabit Network Connection 1521' if=ens9f3 drv=igb unused=vfio-pci
0000:81:00.0 '82599ES 10-Gigabit SFI/SFP+ Network Connection 10fb' if=ens4f0 drv=ixgbe unused=vfio-pci
0000:81:00.1 '82599ES 10-Gigabit SFI/SFP+ Network Connection 10fb' if=ens4f1 drv=ixgbe unused=vfio-pci
0000:83:00.0 '82599ES 10-Gigabit SFI/SFP+ Network Connection 10fb' if=ens7f0 drv=ixgbe unused=vfio-pci
0000:83:00.1 '82599ES 10-Gigabit SFI/SFP+ Network Connection 10fb' if=ens7f1 drv=ixgbe unused=vfio-pci
0000:85:00.0 '82599ES 10-Gigabit SFI/SFP+ Network Connection 10fb' if=ens8f0 drv=ixgbe unused=vfio-pci
0000:85:00.1 '82599ES 10-Gigabit SFI/SFP+ Network Connection 10fb' if=ens8f1 drv=ixgbe unused=vfio-pci
```
Without going again into details, the important points are:
1. Interface ens9f0 is `*Active*` so **do not use it for this application** or you will loose access to the machine
2. The vfio-pci driver (one of the dpdk available drivers) is loaded but not binded to any of the interfaces

So, to bind the driver to the ens8f0 and ens8f1 interfaces we will use this command:
```bash
sudo ./dpdk-devbind.py --bind=vfio-pci 85:00.0 85:00.1 # the address parameter is the PCI address of the desired devices
```
Which will produce:
```
etwork devices using DPDK-compatible driver
============================================
0000:85:00.0 '82599ES 10-Gigabit SFI/SFP+ Network Connection 10fb' drv=vfio-pci unused=ixgbe
0000:85:00.1 '82599ES 10-Gigabit SFI/SFP+ Network Connection 10fb' drv=vfio-pci unused=ixgbe

Network devices using kernel driver
===================================
0000:01:00.0 'I350 Gigabit Network Connection 1521' if=ens9f0 drv=igb unused=vfio-pci *Active*
0000:01:00.1 'I350 Gigabit Network Connection 1521' if=ens9f1 drv=igb unused=vfio-pci
0000:01:00.2 'I350 Gigabit Network Connection 1521' if=ens9f2 drv=igb unused=vfio-pci
0000:01:00.3 'I350 Gigabit Network Connection 1521' if=ens9f3 drv=igb unused=vfio-pci
0000:81:00.0 '82599ES 10-Gigabit SFI/SFP+ Network Connection 10fb' if=ens4f0 drv=ixgbe unused=vfio-pci
0000:81:00.1 '82599ES 10-Gigabit SFI/SFP+ Network Connection 10fb' if=ens4f1 drv=ixgbe unused=vfio-pci
0000:83:00.0 '82599ES 10-Gigabit SFI/SFP+ Network Connection 10fb' if=ens7f0 drv=ixgbe unused=vfio-pci
0000:83:00.1 '82599ES 10-Gigabit SFI/SFP+ Network Connection 10fb' if=ens7f1 drv=ixgbe unused=vfio-pci
```
We are now ready to run the l2fwd application:
```bash
sudo ./dpdk-l2fwd -l 14-15 -- -p 0x3
```
Lets break down the parameters of this command in two parts separated by `--`. The first part are the Environment Abstraction Layer (EAL) arguments, which tell dpdk on which cores to run this application. Further parameters exist but will not be covered here. And the second part are the application parameters. In this case as we use 2 ports we provide a hexadecimal mask (0x3=0b11) telling dpdk that we want to use the 2 ports we have binded.

The application execution will dump a lot of environment information such as the scan of ports and whether they will be binded to the application or not. And finally it will enter into a loop which will provide packet statistics every couple seconds:
```
Lcore 14: RX port 0 TX port 1
Lcore 15: RX port 1 TX port 0
Initializing port 0... done:
Port 0, MAC address: 90:E2:BA:86:61:94
Initializing port 1... done:
Port 1, MAC address: 90:E2:BA:86:61:95

Checking link statusdone
Port 0 Link up at 10 Gbps FDX Autoneg
Port 1 Link up at 10 Gbps FDX Autoneg
L2FWD: entering main loop on lcore 15
L2FWD:  -- lcoreid=15 portid=1
L2FWD: entering main loop on lcore 14
L2FWD:  -- lcoreid=14 portid=0

Port statistics ====================================
Statistics for port 0 ------------------------------
Packets sent:                        0
Packets received:                    0
Packets dropped:                     0
Statistics for port 1 ------------------------------
Packets sent:                        0
Packets received:                    0
Packets dropped:                     0
Aggregate statistics ===============================
Total packets sent:                  0
Total packets received:              0
Total packets dropped:               0
====================================================
```
Until we stop the application with the stop signal: Ctrl+C, it will forward all packets from port 0 to port 1 and vice-versa.

I have used a packet generator and produced some IPmix traffic with these results:
```
Port statistics ====================================
Statistics for port 0 ------------------------------
Packets sent:                 25702050
Packets received:                    2
Packets dropped:                     0              
Statistics for port 1 ------------------------------
Packets sent:                        2
Packets received:             25702089
Packets dropped:                     0
Aggregate statistics ===============================
Total packets sent:           25702084
Total packets received:       25702110
Total packets dropped:               0
====================================================
```

### 7. Clean up the dpdk setup
As the interfaces are now binded to the user-space, they are not visible to the kernel and commands like
```bash
ip -br link
```
will not list them.
To bind them again to the kernel driver we will use the same script `dpdk-devbind.py` to bind them again to their original driver `ixgbe`:
```bash
sudo python3 ../../usertools/dpdk-devbind.py -b ixgbe 85:00.0 85:00.1
```
We can now verify that they are again visible to the kernel:
```bash
ip -br link | grep ens8f
```
Which shows:
```
ens8f0           UP             90:e2:ba:86:61:94 <BROADCAST,MULTICAST,UP,LOWER_UP>
ens8f1           UP             90:e2:ba:86:61:95 <BROADCAST,MULTICAST,UP,LOWER_UP>
```
