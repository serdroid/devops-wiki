# How to set processor affinity on a KVM
On the [NUMA](https://en.wikipedia.org/wiki/Non-uniform_memory_access) systems process migration is usually an expensive operation. To avoid excessive migration of KVM processes we can set processor affinity ( _a.k.a. processor pinning_ ). Otherwise, this could lead us to performance-related problems.
## Identify CPU and NUMA topology
We can use `lscpu` command to identify processor and NUMA topology of a system.

```
root@test0:~# lscpu 
Architecture:        x86_64
CPU op-mode(s):      32-bit, 64-bit
Byte Order:          Little Endian
Address sizes:       46 bits physical, 48 bits virtual
CPU(s):              28
On-line CPU(s) list: 0-27
Thread(s) per core:  1
Core(s) per socket:  14
Socket(s):           2
NUMA node(s):        2
Vendor ID:           GenuineIntel
CPU family:          6
Model:               63
Model name:          Intel(R) Xeon(R) CPU E5-2695 v3 @ 2.30GHz
Stepping:            2
CPU MHz:             2066.313
CPU max MHz:         3300.0000
CPU min MHz:         1200.0000
BogoMIPS:            4594.54
Virtualization:      VT-x
L1d cache:           32K
L1i cache:           32K
L2 cache:            256K
L3 cache:            35840K
NUMA node0 CPU(s):   0-6,14-20
NUMA node1 CPU(s):   7-13,21-27
Flags:               fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush dts acpi mmx fxsr sse sse2 ss ht tm pbe syscall nx pdpe1gb rdtscp lm constant_tsc arch_perfmon pebs bts rep_good nopl xtopology nonstop_tsc cpuid aperfmperf pni pclmulqdq dtes64 monitor ds_cpl vmx smx est tm2 ssse3 sdbg fma cx16 xtpr pdcm pcid dca sse4_1 sse4_2 x2apic movbe popcnt aes xsave avx f16c rdrand lahf_lm abm cpuid_fault epb invpcid_single pti intel_ppin tpr_shadow vnmi flexpriority ept vpid ept_ad fsgsbase tsc_adjust bmi1 avx2 smep bmi2 erms invpcid cqm xsaveopt cqm_llc cqm_occup_llc dtherm ida arat pln pts
```
We see that this system has two NUMA nodes.
```
NUMA node0 CPU(s):   0-6,14-20
NUMA node1 CPU(s):   7-13,21-27
```

## Set CPU Affinity
We can edit the domain (VM) to set an offline domain. We should set cpuset attribute on vcpu definition. See example below:
```
virsh edit $dom
<vcpu placement='static' cpuset='7-13,21-27'>2</vcpu>
```
 
If we want to change the cpu affinity of an online domain we can use `virsh vcpupin <domain> <vcpu> <cpuset>` command. We can view current status with `virsh vcpuinfo <domain>` command. 
```
virsh vcpuinfo $dom
VCPU:           0
CPU:            10
State:          running
CPU time:       3326.4s
CPU Affinity:   yyyyyyyyyyyyyyyyyyyyyyyyyyyy
``` 

We can use below command to set the 7th to 13th and 21st to 27th CPUs to first (0 indexed) vcpu of the domain. If domain has multiple vcpus all vcpus should be set.
```
 virsh vcpupin $dom 0 7-13,21-27
 ```
 
After setting cpu affinity with virsh vcpupin command, we can see that cpu affinity has changed.
 ```

 virsh vcpuinfo $dom
VCPU:           0
CPU:            22
State:          running
CPU time:       4255.1s
CPU Affinity:   -------yyyyyyy-------yyyyyyy

 ```
 
 
 
 ## Set CPU affinity for a group of VMs

When we have a group of KVMs to set cpu affinity, we can make a file that contains domain names one per line and use the script below. On the first line of the while loop we get vcpu count of the domain. And then set all vcpus cpu affinity to desired cpuset in a for loop.

```
while read dom
do 
  cnt=$(virsh dominfo $dom | sed -e '/CPU(/!d' -e 's/.*\s\([0-9]*$\)/\1/')
  for (( ix=0; ix < $cnt; ix++ ))
  do 
    virsh vcpupin $dom $ix 1-6,14-20
  done
done < worker-vms | bash
```

