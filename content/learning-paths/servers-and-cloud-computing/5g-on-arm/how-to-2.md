---
title: Setup and Configure 5G Arm Servers
weight: 3

### FIXED, DO NOT MODIFY
layout: learningpathall
---

## How to Setup and Configure 5G Servers
In this section you get an overview of the different settings used to configure your Arm server. 

### Firmware Maintenance

The BIOS (Basic Input/Output System) firmware impacts some of the functionality and performance of the server. So it is important to sign up for firmware updates from your server vendors.

### How to update firmware
Lets look at the different ways you can update the firmware on your server.

#### Via Web UI (User Interface)

Most of Arm servers provide web UI for managing the firmware (FW).

Log into the Baseboard Management Controller (BMC) IP address via your browser, 

   1. Check the current FW version by clicking its `Firmware information`.

   2. Go to the `Firmware maintenance` section and click on `Firmware update`. Follow the prompts to update BIOSP and BMC software.


#### Via ipmitool command line

The benefit of using the Intelligent Platform Management Interface (IPMI) tool `ipmitool` is for automation of controlling the servers remotely.

 For example on the Foxconn server, run the following commands to update the firmware:

```console
   ipmitool -C 3 -I lanplus -H 10.118.45.98 -U admin -P admin raw 0x32 0x8f 0x3 0x1
   ipmitool -C 3 -I lanplus -H 10.118.45.98 -U admin -P admin -z 8196 hpm upgrade MtCollinsBmc0_43_4.hpm force
   ipmitool -C 3 -I lanplus -H 10.118.45.98 -U admin -P admin raw 0x32 0x8f 0x3 0x1
   ipmitool -C 3 -I lanplus -H 10.118.45.98 -U admin -P admin -z 8196 hpm upgrade 0ACOD009.hpm force
   ipmitool -C 3 -I lanplus -H 10.118.45.98 -U admin -P admin -z 8196 hpm upgrade Mt.Collins_MB_CPLD_v06_20220607.hpm force
   ipmitool -C 3 -I lanplus -H 10.118.45.98 -U admin -P admin -z 8196 hpm upgrade altra_scp_signed_2.10.20220531.hpm force
```

{{% notice Note %}}
The update of the firmware via IPMI interface is hardware dependent, check your server ODM/OEM (Original Design/Equipment Manufacturer) for the IPMI detail for updating its firmware.
{{% /notice %}}

### BIOS Setting

Typically, Single Root IO Virtualization (SR-IOV) enablement is required for Arm servers to support 5G deployment in a container environment. Through BIOS options, you can enable SR-IOV.

### PCIe Setting

This setting is dependent on the server ODM, contact the ODM/OEM to find out details.

### CPU Frequency Setting

The following script can be run to temporarily force the cores to run at max CPU frequency:

```console
#!/bin/bash
  
for ((i=0;i<`nproc`;i++))
do
  {
    #echo ondemand > /sys/devices/system/cpu/cpu$i/cpufreq/scaling_governor
    #cat /sys/devices/system/cpu/cpu$i/cpufreq/cpuinfo_min_freq /sys/devices/system/cpu/cpu$i/cpufreq/scaling_min_freq
    cat /sys/devices/system/cpu/cpu$i/cpufreq/cpuinfo_max_freq  > /sys/devices/system/cpu/cpu$i/cpufreq/scaling_min_freq
    echo performance > /sys/devices/system/cpu/cpu$i/cpufreq/scaling_governor
  }
done
  
for ((i=0;i<=79;i++))
do
  {
    cat /sys/devices/system/cpu/cpu$i/cpufreq/cpuinfo_cur_freq
  }
done
```

#### Change frequency scaling

Typically there is a way in the BIOS to disable frequency scaling. There's also a Linux option, described below: 

```console
cat /sys/devices/system/cpu/cpufreq/policy0/scaling_governor
```
This command returns `ondemand` in the current setup.

You may want to change the CPU governor from `ondemand` to `performance`.

The following commands show you how to do it:

```console
sudo apt install cpufrequtils
echo 'GOVERNOR="performance"' | sudo tee /etc/default/cpufrequtils
sudo systemctl disable ondemand
```

### Linux Requirements

#### Boot Parameters

The individual Linux kernel command-line parameters that impact running the 5G stack are described below:

##### Hugepage setting
default_hugepagesz=1G hugepagesz=1G hugepages=32

The huge page setting is required for DPDK (Data Plane Development Kit) support which is commonly adopted in 5G networking. It reduces the TLB (Translation Lookaside Buffer) misses, improves performance by lowering memory overhead, and improves memory efficiency with larger and contiguous memory allocation.

##### isolcpus=cpuX-cpuY
The setting "isolcpus" is a kernel command line parameter that is used to isolate specific CPUs in a multi-CPU system, such that they are not used for general purpose processing. Instead, they are reserved for real-time or other special purpose tasks that require dedicated and predictable processing resources.

The "isolcpus" setting is specified as a list of CPU numbers, separated by commas. For example, if you wanted to isolate CPUs 0 and 1, you would specify "isolcpus=0,1" on the kernel command line.

By isolating specific CPUs, it is possible to ensure that real-time tasks or other critical processes have access to dedicated and predictable processing resources, without being impacted by general purpose processing or other system tasks. This can be useful in a variety of applications, including real-time systems, virtualization, and high-performance computing.

##### nohz_full=cpuX-cpuY
The setting "nohz_full" is a kernel command line parameter that is used to control the behavior of the Linux kernel's tickless system feature. The tickless system feature is used to reduce the amount of time that the CPU spends executing the kernel's timer interrupt, which is known as the "tick".

The "nohz_full" setting is used to specify a list of CPU numbers that should be designated as "full tick" CPUs. Full tick CPUs are CPUs that continue to receive the tick even when the system is idle.

When the tickless system feature is enabled, the CPU will stop executing the tick on all but the full tick CPUs when the system is idle. This can reduce the amount of CPU time that is spent executing the tick, which can result in reduced power consumption and improved performance for certain workloads.

##### rcu_nocbs=cpuX-cpuY
The "rcu_nocbs" setting is used to specify a list of CPU numbers that should be designated as no-callback (nocb) CPUs. Nocb CPUs are CPUs that are used exclusively for RCU read-side critical sections, and are not used for callback processing.

By designating specific CPUs as nocb CPUs, it is possible to reduce the overhead of callback processing, as callback processing can be a significant source of overhead for the RCU subsystem. Additionally, it can also reduce lock contention, as callback processing often requires the use of locks.

##### rcu_nocb_poll
The setting "rcu_nocb_poll" is a kernel command line parameter that is used to control the behavior of the RCU (Read-Copy Update) subsystem in the Linux kernel. The RCU subsystem is a mechanism used to safely update shared data structures in a multi-threaded environment, without the need for locks or other synchronization mechanisms.

The "rcu_nocb_poll" setting is used to enable or disable polling for callbacks from the RCU subsystem in the no-callback (nocb) CPUs. When this setting is set to "1", polling is enabled, and when it's set to "0", polling is disabled.

Enabling polling for callbacks in the nocb CPUs can provide a performance improvement for some workloads, as it reduces the amount of time that the RCU subsystem spends waiting for callbacks to be processed. However, this performance improvement may come at the cost of increased CPU utilization, as polling for callbacks can consume additional CPU cycles.

It's important to carefully consider the impact of the "rcu_nocb_poll" setting before using it in a production environment, as the performance impact can vary depending on the specific workload and system configuration. Additionally, it's recommended to thoroughly test the setting in a controlled environment before deploying it in a production setting.

##### irqaffinity=cpuX-cpuY
irqaffinity is a kernel parameter in Linux that sets the CPU affinity for IRQs (Interrupt Requests). IRQs are a mechanism used by the kernel to handle hardware interrupts, which are signals generated by devices to indicate that they need attention.

By setting irqaffinity, you can specify which CPU or CPUs should handle a specific IRQ. This can be useful for optimizing system performance and avoiding IRQ-related bottlenecks.

The value for irqaffinity= is a list of CPUs, separated by commas, that should handle a specific IRQ. The IRQ number can be specified either in decimal or hexadecimal format, depending on your system.

##### nosoftlockup
The setting "nosoftlockup" is a kernel command line parameter that is used to control the behavior of the Linux kernel's software lockup detection mechanism.

A software lockup is a situation where the kernel is unresponsive and cannot process new tasks or respond to system calls. This can occur if a task enters an infinite loop, causing the system to hang. The software lockup detection mechanism is used to detect and recover from such situations.

The "nosoftlockup" setting disables the software lockup detection mechanism. When this setting is used, the kernel will not detect software lockups and will not attempt to recover from them.

It's important to carefully consider the impact of the "nosoftlockup" setting before using it in a production environment. Disabling the software lockup detection mechanism may result in the system hanging or becoming unresponsive if a software lockup occurs. This can cause data loss, corruption, or other serious problems. Additionally, it can make it more difficult to diagnose and resolve software lockup issues. As a result, it's generally recommended to leave the software lockup detection mechanism enabled, unless there is a specific reason to disable it.

##### iommu.passthrough=1
The "iommu.passthrough=1" setting is used to enable IOMMU (Input–Output Memory Management Unit) pass-through, which allows virtual machines to access physical devices without any virtualization of the I/O memory.

This setting is used in virtualization environments where it is important to have high performance I/O for certain applications or devices. For example, some graphics-intensive applications require direct access to the physical GPU, which can be achieved by enabling IOMMU pass-through. However, enabling IOMMU pass-through can also make the virtualization environment less secure, as it provides direct access to physical devices, bypassing any virtualization or security features. As a result, the setting is typically used with caution, and only in specific cases where the performance benefits outweigh the security risks. 

#####  cpufreq.off=1  
The setting "cpufreq.off=1" is a kernel command line parameter that is used to disable the CPU frequency scaling feature in the Linux kernel. CPU frequency scaling, also known as CPU speed scaling, is a technique that allows the operating system to dynamically adjust the clock speed of the CPU based on the workload. This can help to conserve energy and reduce heat generation, while still providing sufficient performance for the current workload.

The "cpufreq.off=1" setting is used to disable this feature, which can be useful in certain scenarios where it is important to have a consistent and predictable CPU performance. For example, some real-time or embedded systems may require a fixed CPU clock speed to ensure consistent and predictable performance, which can be achieved by disabling CPU frequency scaling.

It's important to note that disabling CPU frequency scaling can have a negative impact on energy efficiency, as the CPU will run at the highest clock speed at all times, regardless of the workload. Additionally, it can also generate more heat, which can be a concern in some systems. As a result, it's important to carefully consider the impact of this setting before disabling CPU frequency scaling in a production environment.

##### kpti=off
"kpti=off" is a kernel parameter that can be used to disable the Kernel Page Table Isolation (KPTI) feature in Linux operating systems.

KPTI is a security feature that was introduced to address the Spectre and Meltdown vulnerabilities, which allowed attackers to access sensitive data on a system by exploiting a design flaw in modern processors. When KPTI is enabled, it isolates the kernel's page tables from user-space page tables to prevent such attacks.

However, some older processors, particularly those with limited address space, can experience performance issues when KPTI is enabled. In such cases, the "kpti=off" parameter can be used to disable KPTI and potentially improve system performance.

It is important to note that disabling KPTI can leave the system vulnerable to Spectre and Meltdown attacks, so it should only be done with caution and after assessing the specific risks and benefits for a particular system.

##### modprobe.blacklist=cppc_cpufreq
modprobe.blacklist=cppc_cpufreq is a kernel parameter in Linux that blacklists the cppc_cpufreq module. A module in Linux is a piece of code that can be loaded and unloaded dynamically into the kernel to provide additional functionality.

By blacklisting the cppc_cpufreq module, you are telling the Linux kernel not to load it at boot time. This means that the functionality provided by the module will not be available to the system.

##### tsc=reliable 
The setting "tsc=reliable" is a kernel command line parameter that is used to set the Time Stamp Counter (TSC) as a reliable source of time for the Linux kernel. The TSC is a CPU register that increments at a fixed rate based on the clock speed of the CPU. It is commonly used as a source of time for the operating system, as it provides a high-resolution and constant-rate timer that is usable even when other system timers are unavailable or unreliable.

The "tsc=reliable" setting is used to indicate that the TSC is a reliable source of time, and that it should be used as the primary source of time for the Linux kernel. This can be useful in some cases where the TSC is known to be reliable and accurate, as it can provide more consistent and accurate timing information for the operating system.

It's important to note that the TSC may not be reliable or accurate on all systems, as it depends on the underlying hardware and other system configurations. For example, some multi-socket systems may have processors with different clock speeds, which can result in inconsistent TSC values. Additionally, some processors may have bugs that affect the accuracy of the TSC, or the TSC may be reset or stop counting under certain conditions. As a result, it's important to carefully consider the reliability and accuracy of the TSC before using the "tsc=reliable" setting in a production environment.

##### clocksource=tsc
The setting "clocksource=tsc" is a kernel command line parameter that is used to set the time stamp counter (TSC) as the clock source for the Linux kernel. The TSC is a CPU register that increments at a fixed rate based on the clock speed of the CPU. It is commonly used as a source of time for the operating system, as it provides a high-resolution and constant-rate timer that is usable even when other system timers are unavailable or unreliable.

The "clocksource=tsc" setting is used to indicate that the TSC should be used as the clock source for the Linux kernel. This can be useful in some cases where the TSC is known to be reliable and accurate, as it can provide more consistent and accurate timing information for the operating system.

It's important to note that the TSC may not be reliable or accurate on all systems, as it depends on the underlying hardware and other system configurations. For example, some multi-socket systems may have processors with different clock speeds, which can result in inconsistent TSC values. Additionally, some processors may have bugs that affect the accuracy of the TSC, or the TSC may be reset or stop counting under certain conditions. As a result, it's important to carefully consider the reliability and accuracy of the TSC before using the "clocksource=tsc" setting in a production environment.

##### cma=on cma=###m
cma=on is a kernel parameter in Linux that enables the Contiguous Memory Allocator (CMA) feature. CMA is a feature that allows the kernel to allocate large contiguous blocks of memory for use by device drivers. This can be useful for certain types of devices, such as graphics cards, that require large, contiguous memory allocations.

By setting cma=on, you are telling the Linux kernel to use the CMA feature. This can improve the performance and functionality of certain types of devices that require large, contiguous memory allocations.

cma=###m such as cma=256m is to ask the kernel to allocate 256 MB for CMA memory during boot

##### audit=0
audit=0 is a kernel parameter in Linux that disables the audit system. The audit system is a feature in Linux that provides a way to track system events, such as user logins, file access, and process execution. The audit system logs these events to the audit log, which can be used for security and troubleshooting purposes.

By setting audit=0, you are telling the Linux kernel not to use the audit system. This means that the audit log will not be generated and the events that would normally be logged will not be tracked.

It's worth noting that disabling the audit system is generally not recommended, as it reduces the visibility into system events and makes it harder to detect and diagnose security problems and other issues. In most cases, it's better to keep the audit system enabled and use appropriate audit rules to control what events are logged.

### Low Latency Kernel

A low latency kernel is a version of the Linux operating system that has been optimized for real-time applications. The goal of a low latency kernel is to minimize the time it takes for the operating system to respond to events and processes, so that applications can run with minimal delay or interruption.

Low latency kernels are commonly used in applications that require real-time processing, such as audio and video production, gaming, and scientific computing, of course in 5G space as well. They are also used in environments where the time taken by the operating system to respond to events is critical, such as in financial trading systems and other high-frequency data processing systems.

To achieve low latency, low latency kernels typically implement a number of changes to the standard Linux kernel, such as reducing the frequency of interrupts, minimizing the number of context switches, and reducing the amount of time spent processing system calls. They may also make use of specialized scheduling algorithms and memory management techniques to further optimize performance.

To install a low latency kernel on Arm server, run the following commands:

```console
sudo apt update
sudo apt-cache policy 'linux-image-5.*-lowlatency'
sudo apt install linux-headers-5.15.0-46-lowlatency linux-image-5.15.0-46-lowlatency -y
```
Then run this following script to verify the new low-latency kernel:

```console
awk -F\' '$1=="menuentry " {print i++ " : " $2} $1=="submenu " {print i++ " : " $2; j=0} $1=="\tmenuentry " {print "\t", j++ " : " $2}' /boot/grub/grub.cfg
```
The output from this command will look like:

```output
         0 : Ubuntu, with Linux 5.15.0-46-lowlatency
         1 : Ubuntu, with Linux 5.15.0-46-lowlatency (recovery mode)
```

You will also need to disable upgrade OS kernel so the kernel will stay at the same version:

```console
sudo vi /etc/apt/apt.conf.d/20auto-upgrades, change 1 to 0:
APT::Periodic::Update-Package-Lists "0";
APT::Periodic::Unattended-Upgrade "0";
```

### Real-Time (RT) Kernel

An RT kernel is designed to provide a guaranteed minimum response time for certain system events and processes, even under heavy load conditions. This is achieved through a number of optimizations to the standard Linux kernel, such as reducing the frequency of interrupts, minimizing the number of context switches, and reducing the amount of time spent processing system calls. Additionally, RT kernels may make use of specialized scheduling algorithms and memory management techniques to further optimize performance.

At this moment, RT kernel is not ready from apt repository, you will need to rebuild the kernel to enable RT for Arm. 

### SR-IOV

#### Supported SR-IOV NICs (Network Interface Cards)

  Intel® Ethernet 800 Series (E810)

  Intel® Ethernet 700 Series (XL710, X710, XXV710)

  Intel® Ethernet 500 Series (82599, X520, X540, X550)

  Mellanox ConnectX-4®

  Mellanox Connectx-4® Lx EN Adapter

  Mellanox ConnectX-5®

  Mellanox ConnectX-5® Ex

  Mellanox ConnectX-6®

  Mellanox ConnectX-6® Dx

  Mellanox BlueField-2®

####  Creating SR-IOV Resources

  Creating SR-IOV Virtual Functions (VFs)

  Generation of SR-IOV configuration maps for Kubernetes
  
       - VM (Virtual Machine) based (K8s configuration map for corresponding worker node)
       - Bare-Metal (SR-IOV VFs and Configuration map)

  Example of SR-IOV configuration map for Kubernetes:
```console
insertModules:
  - "i40e"
  - "vfio-pci"
#pci details of the root device used and corresponding vf mappings
pciConfig:
  - pci: "0000:00:0a.0" #sriov vf pci address which is dev passthrough to VM directly
    targetPfDriver: "iavf"
    vfs: 0  #vf creation is not needed since the pci itself is a sriov VF
  - pci: "0000:00:0b.0" #sriov vf pci address which is dev passthrough to VM directly
    targetPfDriver: "iavf"
    vfs: 0 #vf creation is not needed since the pci itself is a sriov VF
 
resourceConfig:
  resourceList:
    - resourceName: intel_sriov_netdevice_upf_ngu
      resourcePrefix: arm.com
      selectors:
        rootDevices:
          - '0000:00:0a.0'
    - resourceName: intel_sriov_netdevice_upf_n6
      resourcePrefix: arm.com
      selectors:
        rootDevices:
          - '0000:00:0b.0'
```

|Field|Required|Description|Type/Defaults|Example/ACeepted Values|
| ----------- | ----------- | ----------- | ----------- | ----------- |
|"resourceName"|	Y|	Endpoint resource name. Should not contain special characters including hyphens and must be unique in the scope of the resource prefix	|string	|"sriov_net_A"|
|"resourcePrefix"|	N|	Endpoint resource prefix name override. Should not contain special characters	|string Default : "arm.com"	|"yourcompany.com"|
|"selectors"|	N|	A map of device selectors. The "deviceType" value determines the "selectors" options.	|json object as string Default: null	|Example: "selectors": {"vendors": ["8086"],"devices": ["154c"]}|
|"rootDevices"|	N	|functions from PF matches list of PF PCI addresses	|string list Default: null	|"rootDevices": ["0000:86:00.0"] (See follow-up sections for some advance usage of "rootDevices")|

#### Configure Device Plugin extended selectors in virtual environments

SR-IOV Network Device Plugin supports running in a virtualized environment. However, not all device selectors are applicable as the VFs (Virtual Functions) are passthrough to the VM (Virtual Machine) without any association to their respective PF (Physical Function), hence any device selector that relies on the association between a VF and its PF will not work and therefore the pfNames and rootDevices extended selectors will not work in a virtual deployment. The common selector pciAddress can be used to select the virtual device.

#### Configuring SR-IOV Resources
Perform the following steps on the bare-metal host for enabling SR-IOV.

- Enable SR-IOV in BIOS settings.

- Append iommu.passthrough=1 option to GRUB_CMLINE_LINUX line in /etc/default/grub

- Run the `grub-mkconfig -o /boot/grub/grub.cfg` script to update boot partitions grub configuration file.

Reboot Linux to reflect the changes.

#### Type I - SR-IOV Configuration for VM Deployment
echo 2 > /sys/class/net/<interface name>/device/sriov_numvfs

Execute the following command to check if the SR-IOV VFs interface names are displayed:

```console
lshw -c network -businfo
```
Execute the following command to remove the SR-IOV from the bare-metal host systems.

```console
echo 0 > /sys/class/net/<interface name>/device/sriov_numvfs
```
Type II - SR-IOV Configuration for Bare-metal Deployment
Example of a SR-IOV deployment yaml file used for UPF (User Plane Function):

```console
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
    deployment.kubernetes.io/revision: "1"
    meta.helm.sh/release-name: upfnf
    meta.helm.sh/release-namespace: radisys-upf1
  creationTimestamp: "2023-04-28T09:51:46Z"
  generation: 1
  labels:
    app: upf
    app.kubernetes.io/managed-by: Helm
    svc: upf
  name: upf
  namespace: radisys-upf1
  resourceVersion: "11234556"
  uid: 1c98264a-93bf-4ac7-b807-5187779495dc
spec:
  progressDeadlineSeconds: 600
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: upf
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      annotations:
        k8s.v1.cni.cncf.io/networks: '[ { "name": "upf1-ngu-interface", "interface":
          "net1" }, { "name": "upf1-n6-interface", "interface": "net2" } ]'
        myLivenessPath: /fgc-livez
        myProbePort: "8090"
        myReadinessPath: /fgc-readyz
      creationTimestamp: null
      labels:
        app: upf
    spec:
      containers:
      - command:
        - /bin/bash
        - -c
        - sysctl -w net.ipv6.conf.all.disable_ipv6=0;sysctl -w net.ipv4.conf.all.arp_ignore=1;/root/bin/upf
        env:
        - name: INTF_NAME
          value: eth0
        - name: LOG_OUTPUT
          value: STDOUT
        - name: MY_POD_NAME
          valueFrom:
            fieldRef:
              apiVersion: v1
              fieldPath: metadata.name
        - name: MY_POD_NAMESPACE
          valueFrom:
            fieldRef:
              apiVersion: v1
              fieldPath: metadata.namespace
        - name: MY_POD_IP
          valueFrom:
            fieldRef:
              apiVersion: v1
              fieldPath: status.podIP
        - name: MY_POD_LIVENESS_PATH
          valueFrom:
            fieldRef:
              apiVersion: v1
              fieldPath: metadata.annotations['myLivenessPath']
        - name: MY_POD_READINESS_PATH
          valueFrom:
            fieldRef:
              apiVersion: v1
              fieldPath: metadata.annotations['myReadinessPath']
        - name: MY_POD_PROBE_PORT
          valueFrom:
            fieldRef:
              apiVersion: v1
              fieldPath: metadata.annotations['myProbePort']
        image: docker.io/library/upfsp:4.0.2-upfarm
        imagePullPolicy: IfNotPresent
        livenessProbe:
          failureThreshold: 3
          httpGet:
            path: /fgc-livez
            port: 8090
            scheme: HTTP
          initialDelaySeconds: 30
          periodSeconds: 5
          successThreshold: 1
          timeoutSeconds: 1
        name: upfsp
        readinessProbe:
          failureThreshold: 3
          httpGet:
            path: /fgc-readyz
            port: 8090
            scheme: HTTP
          initialDelaySeconds: 30
          periodSeconds: 5
          successThreshold: 1
          timeoutSeconds: 1
        resources:
          limits:
            cpu: "1"
            hugepages-1Gi: 2Gi
            memory: 1Gi
          requests:
            cpu: 500m
            hugepages-1Gi: 2Gi
            memory: 1Gi
        securityContext:
          privileged: true
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        volumeMounts:
        - mountPath: /etc/podinfo
          name: podinfo
        - mountPath: /dev/shm
          name: shm
        - mountPath: /dev/hugepages
          name: hugepage
        - mountPath: /run/vpp
          name: upf-memif
        - mountPath: /root/config
          name: oam-config-volume
        - mountPath: /etc/fgc/upf
          name: day0-config-volume
        - mountPath: /var/local/core-dumps
          name: core-path
      - command:
        - /bin/bash
        - -c
        - sysctl -w net.ipv6.conf.all.autoconf=0;sysctl -w net.ipv6.conf.default.autoconf=0;/opt/upf_fp/bin/upffpmgr
        env:
        - name: INTF_NAME
          value: eth0
        - name: nguIntType
          value: sriov
        - name: n6IntType
          value: sriov
        - name: nguResourcePrefix
          value: intel.com
        - name: nguResourceName
          value: sriov_netdevice_ngu_upf
        - name: nguPciDeviceName
          value: PCIDEVICE_INTEL_COM_SRIOV_NETDEVICE_NGU_UPF
        - name: n6ResourcePrefix
          value: intel.com
        - name: n6ResourceName
          value: sriov_netdevice_n6_upf
        - name: n6PciDeviceName
          value: PCIDEVICE_INTEL_COM_SRIOV_NETDEVICE_N6_UPF
        - name: LOG_OUTPUT
          value: STDOUT
        - name: LD_LIBRARY_PATH
          value: /opt/upf_fp/vpp/lib/:/lib/x86_64-linux-gnu
        - name: PATH
          value: /usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/opt/upf_fp/vpp/bin
        image: docker.io/library/upffp:4.0.2-upfarm
        imagePullPolicy: IfNotPresent
        name: upffp
        resources:
          limits:
            cpu: "4"
            hugepages-1Gi: 3Gi
            intel.com/sriov_netdevice_n6_upf: "1"
            intel.com/sriov_netdevice_ngu_upf: "1"
            memory: 1Gi
          requests:
            cpu: "4"
            hugepages-1Gi: 3Gi
            intel.com/sriov_netdevice_n6_upf: "1"
            intel.com/sriov_netdevice_ngu_upf: "1"
            memory: 1Gi
        securityContext:
          privileged: true
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        volumeMounts:
        - mountPath: /lib/modules
          name: modules
        - mountPath: /dev/vfio
          name: dev
        - mountPath: /sys/devices
          name: devices
        - mountPath: /sys/bus/pci
          name: pci
        - mountPath: /sys/module
          name: sysmodule
        - mountPath: /etc/podinfo
          name: podinfo
        - mountPath: /run/vpp
          name: upf-memif
        - mountPath: /dev/hugepages
          name: hugepage
        - mountPath: /dev/shm
          name: shm
        - mountPath: /opt/upf_fp/conf
          name: config-volume
        - mountPath: /opt/upf_fp/config
          name: oam-config-volume
        - mountPath: /var/local/core-dumps
          name: core-path
      - command:
        - /bin/bash
        - -c
        - rsyslogd -n
        env:
        - name: INTF_NAME
          value: eth0
        - name: LOG_OUTPUT
          value: STDOUT
        - name: MY_POD_NAME
          valueFrom:
            fieldRef:
              apiVersion: v1
              fieldPath: metadata.name
        - name: MY_POD_NAMESPACE
          valueFrom:
            fieldRef:
              apiVersion: v1
              fieldPath: metadata.namespace
        - name: MY_POD_IP
          valueFrom:
            fieldRef:
              apiVersion: v1
              fieldPath: status.podIP
        image: localhost:5000/upfrsyslog:v1
        imagePullPolicy: IfNotPresent
        name: upfrsyslog
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        volumeMounts:
        - mountPath: /var/log/upfv3
          name: upfv3-syslog
        - mountPath: /etc/rsyslog.d
          name: rsyslog-config-volume
      dnsPolicy: ClusterFirst
      nodeName: node1
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
      volumes:
      - hostPath:
          path: /lib/modules
          type: ""
        name: modules
      - hostPath:
          path: /dev/vfio
          type: ""
        name: dev
      - hostPath:
          path: /sys/devices
          type: ""
        name: devices
      - hostPath:
          path: /sys/bus/pci
          type: ""
        name: pci
      - hostPath:
          path: /sys/module
          type: ""
        name: sysmodule
      - downwardAPI:
          defaultMode: 420
          items:
          - fieldRef:
              apiVersion: v1
              fieldPath: metadata.labels
            path: labels
          - fieldRef:
              apiVersion: v1
              fieldPath: metadata.annotations
            path: annotations
        name: podinfo
      - emptyDir: {}
        name: upf-memif
      - hostPath:
          path: /dev/shm/
          type: ""
        name: shm
      - emptyDir:
          medium: HugePages
        name: hugepage
      - configMap:
          defaultMode: 420
          name: upf-radisys-upf1-configmap
        name: config-volume
      - configMap:
          defaultMode: 420
          name: upf-radisys-upf1-oam-configmap
        name: oam-config-volume
      - configMap:
          defaultMode: 420
          name: upf-radisys-upf1-day0-configmap
        name: day0-config-volume
      - configMap:
          defaultMode: 420
          name: upf-radisys-upf1-rsyslog-configmap
        name: rsyslog-config-volume
      - hostPath:
          path: /var/local/core-dumps
          type: ""
        name: core-path
      - hostPath:
          path: /var/log/upfv3
          type: ""
        name: upfv3-syslog
status:
  conditions:
  - lastTransitionTime: "2023-04-28T09:51:46Z"
    lastUpdateTime: "2023-04-28T09:51:46Z"
    message: Deployment does not have minimum availability.
    reason: MinimumReplicasUnavailable
    status: "False"
    type: Available
  - lastTransitionTime: "2023-04-28T10:01:47Z"
    lastUpdateTime: "2023-04-28T10:01:47Z"
    message: ReplicaSet "upf-5bf69d8f7c" has timed out progressing.
    reason: ProgressDeadlineExceeded
    status: "False"
    type: Progressing
  observedGeneration: 1
  replicas: 1
  unavailableReplicas: 1
  updatedReplicas: 1
```

```console
git clone https://github.com/k8snetworkplumbingwg/sriov-cni
kubectl apply -f sriov-cni/images/k8s-v1.16/sriov-cni-daemonset.yaml
```

sriov-cni-daemonset.yaml:
```console
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: kube-sriov-cni-ds-arm64
  namespace: kube-system
  labels:
    tier: node
    app: sriov-cni
spec:
  selector:
    matchLabels:
      name: sriov-cni
  template:
    metadata:
      labels:
        name: sriov-cni
        tier: node
        app: sriov-cni
    spec:
      nodeSelector:
        kubernetes.io/arch: arm64
      tolerations:
      - key: node-role.kubernetes.io/master
        operator: Exists
        effect: NoSchedule
      containers:
      - name: kube-sriov-cni-arm64
        image: localhost:5000/sriov-cni-arm64:latest
        imagePullPolicy: IfNotPresent
        securityContext:
          allowPrivilegeEscalation: false
          privileged: false
          readOnlyRootFilesystem: true
          capabilities:
            drop:
              - ALL
        resources:
          requests:
            cpu: "100m"
            memory: "50Mi"
          limits:
            cpu: "100m"
            memory: "50Mi"
        volumeMounts:
        - name: cnibin
          mountPath: /host/opt/cni/bin
      volumes:
        - name: cnibin
          hostPath:
            path: /opt/cni/bin
```

### SR-IOV CNI Resource Verification
Perform the following step to verify the SR-IOV CNI (Cloud Native Interface) resource configuration.

Execute the following command in the master node and confirm if allocatable resources were created on the worker node used for UPF deployment:

```console
kubectl get node <node name> -o json | jq '.status.allocatable'
```
This command displays the following sample output:
```console
{
    "cpu": "31900m",
    "ephemeral-storage": "189274027310",
    "hugepages-1Gi": "10Gi",
    "memory": "120754964Ki",
    "pods": "110",
    "arm.com/intel_sriov_netdevice_upf_n6": "1",
    "arm.com/intel_sriov_netdevice_upf_ngu": "1"
}
```

### PTP (Precision Time Protocol) Setting
In 5G world, the time synchronization is critical to synchronize all components in same timing cadence.   

#### GrandMaster Hardware based PTP

This option requires a GrandMaster Qulsar with GPS capability

Also requires a PTP enabled switch, currently Arista switch has PTP sync'ed to all slave nodes like Nvidia/Keysight.

#### SW based PTP

How to setup ptp4l/phc2sys on Linux:

Configure the slave node with the contents of the configuration file shown below:

```console
$ cat /etc/5g-ptp.conf
[global]
verbose 1
domainNumber 24
slaveOnly 1
priority1 128
priority2 128
use_syslog 0
logging_level 6
tx_timestamp_timeout 900
hybrid_e2e 0
dscp_event 46
dscp_general 46
#clock_type BC
boundary_clock_jbod 1
 
[enP1p3s0f0np0]
logAnnounceInterval -3
announceReceiptTimeout 3
logSyncInterval -4
logMinDelayReqInterval -4
delay_mechanism E2E
network_transport L2
```

Setup Slave node PTP4L service: note that you need to assign a core for this task to run

```console
$ cat /lib/systemd/system/ptp4l.service
[Unit]
Description=Precision Time Protocol (PTP) service
Documentation=man:ptp4l
 
[Service]
Type=simple
ExecStart=taskset -c 32 /usr/sbin/ptp4l -f /etc/5g-ptp.conf
 
[Install]
WantedBy=multi-user.target
``` 

Setup Slave node PHC2SYS service: note the NIC interface used here is enP1p3s0f0np0, also assign a core to run it

```console 
$ cat /lib/systemd/system/phc2sys.service
[Unit]
Description=Synchronize system clock or PTP hardware clock (PHC)
Documentation=man:phc2sys
After=ntpdate.service
Requires=ptp4l.service
After=ptp4l.service
 
[Service]
Type=simple
ExecStart=/bin/sh -c "taskset -c 31 /usr/sbin/phc2sys -s /dev/ptp$(ethtool -T enP1p3s0f0np0 | grep PTP | awk '{print $4}') -c CLOCK_REALTIME -n 24 -O 0 -R 256 -u 256"
 
[Install]
WantedBy=multi-user.target
```
 
Configure Master node setting:
```console
$ cat /etc/5g-ptp.conf
[global]
verbose 1
domainNumber 24
priority1 128
priority2 128
use_syslog 1
logging_level 6
tx_timestamp_timeout 30
hybrid_e2e 0
dscp_event 46
dscp_general 46
[ens1f0np0]
logAnnounceInterval -3
announceReceiptTimeout 3
logSyncInterval -4
logMinDelayReqInterval -3
delay_mechanism E2E
network_transport L2
```
 
Setup Master node PTP4L service: note that we need to assign a core for this task to run
```console 
$ cat /lib/systemd/system/ptp4l.service
[Unit]
Description=Precision Time Protocol (PTP) service
Documentation=man:ptp4l
 
[Service]
Type=simple
Restart=always
RestartSec=5s
ExecStart=taskset -c 32 /usr/sbin/ptp4l -f /etc/5g-ptp.conf
 
[Install]
WantedBy=multi-user.target
```
 
Setup Master node PHC2SYS service: note the NIC interface used here is enP1p3s0f0np0, also assign a core to run it
```console
$ cat /lib/systemd/system/phc2sys.service
[Unit]
Description=Synchronize system clock or PTP hardware clock (PHC)
Documentation=man:phc2sys
After=ntpdate.service
Requires=ptp4l.service
After=ptp4l.service
 
[Service]
Type=simple
Restart=always
RestartSec=5s
ExecStart=/bin/sh -c "taskset -c 31 /usr/sbin/phc2sys -s /dev/ptp$(ethtool -T enP1p3s0f0np0 | grep PTP | awk '{print $4}') -c CLOCK_REALTIME -n 24 -O 0 -R 256 -u 256"
 
[Install]
WantedBy=multi-user.target
```
 
Start PTP/PHC2SYS services:

```console 
#PTP.service
sudo systemctl daemon-reload
sudo systemctl restart ptp4l.service
sudo systemctl enable ptp4l.service
sudo systemctl status ptp4l.service
 
#phc2sys.service
sudo systemctl daemon-reload
sudo systemctl restart phc2sys.service
sudo systemctl enable phc2sys.service
sudo systemctl status phc2sys.service
``` 



