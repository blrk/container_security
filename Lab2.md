### Isolation & namespaces
* view namespaces on the host
``` bash
sudo lsns

      NS TYPE   NPROCS   PID USER            COMMAND
4026531834 time      122     1 root            /usr/lib/systemd/systemd --switched-root --system --deserialize=32
4026531835 cgroup    116     1 root            /usr/lib/systemd/systemd --switched-root --system --deserialize=32
4026531836 pid       116     1 root            /usr/lib/systemd/systemd --switched-root --system --deserialize=32
4026531837 user      122     1 root            /usr/lib/systemd/systemd --switched-root --system --deserialize=32
4026531838 uts       109     1 root            /usr/lib/systemd/systemd --switched-root --system --deserialize=32
4026531839 ipc       116     1 root            /usr/lib/systemd/systemd --switched-root --system --deserialize=32
4026531840 net       116     1 root            /usr/lib/systemd/systemd --switched-root --system --deserialize=32
4026531841 mnt       102     1 root            /usr/lib/systemd/systemd --switched-root --system --deserialize=32
4026531862 mnt         1    26 root            kdevtmpfs
4026532277 mnt         1  1311 root            /usr/lib/systemd/systemd-udevd
4026532391 mnt         1  1943 root            /sbin/auditd
4026532392 uts         1  1311 root            /usr/lib/systemd/systemd-udevd
4026532393 mnt         1  1942 systemd-resolve /usr/lib/systemd/systemd-resolved
4026532440 mnt         1  2062 chrony          /usr/sbin/chronyd -F 2
4026532441 mnt         1  2033 root            /usr/sbin/irqbalance --foreground
4026532442 mnt         2  2039 dbus            /usr/bin/dbus-broker-launch --scope system --audit
4026532443 mnt         1  2038 root            /usr/lib/systemd/systemd-logind
4026532444 uts         1  2038 root            /usr/lib/systemd/systemd-logind
4026532445 uts         1  2062 chrony          /usr/sbin/chronyd -F 2
4026532446 mnt         4  2281 root            /usr/lib/systemd/systemd-userdbd
4026532447 uts         4  2281 root            /usr/lib/systemd/systemd-userdbd
4026532513 mnt         1  2099 systemd-network /usr/lib/systemd/systemd-networkd
4026532531 mnt         6 26792 systemd-oom     postgres
4026532532 uts         6 26792 systemd-oom     postgres
4026532533 ipc         6 26792 systemd-oom     postgres
4026532534 pid         6 26792 systemd-oom     postgres
4026532535 net         6 26792 systemd-oom     postgres
4026532604 cgroup      6 26792 systemd-oom     postgres
```
* Note: The “NPROCS” field shows that 122 processes are using the first set of namespaces on this host
* Start a nginx container
``` bash
docker run -d nginx:1.23.1
```
* Re-run the lsns comamnd to see the new set of namespaces for nginx processes
```bash
4026532454 mnt         3 27685 root            nginx: master process nginx -g daemon off;
4026532455 uts         3 27685 root            nginx: master process nginx -g daemon off;
4026532456 ipc         3 27685 root            nginx: master process nginx -g daemon off;
4026532457 pid         3 27685 root            nginx: master process nginx -g daemon off;
4026532458 net         3 27685 root            nginx: master process nginx -g daemon off;
```
#### Mount namespace
* The mount (mnt) namespace provides a process with an isolated view of the filesystem. It can be useful for ensuring that processes don’t interfere with files that belong to other processes on the host.
* You can see which mount namespaces are used by a process by looking in the /proc filesystem; the information is contained in /proc/[PID]/mountinfo
```bash
cat /proc/27685/mountinfo 

682 633 0:48 / / rw,relatime master:294 - overlay overlay rw,seclabel,lowerdir=/var/lib/docker/overlay2/l/EYESVL37XVDGZUJQGXTESSXQVI:/var/lib/docker/overlay2/l/DVLCRI2H5VHRHMQX2VYCARFLPY:/var/lib/docker/overlay2/l/OGEQVQXL2VUVOJ3UWLEHOZWYSL:/var/lib/docker/overlay2/l/ZSHH6FLJS6RG5P2VTFDBAKYPNN:/var/lib/docker/overlay2/l/4MBOQUUKMQIJS6XEBJF5RKKUKG:/var/lib/docker/overlay2/l/JOVZWMZXF37DQKZ2QYXPPOI2CD:/var/lib/docker/overlay2/l/GFBDT6QCLJVJ3SGHCRVBGVHXCD,upperdir=/var/lib/docker/overlay2/ba5316850213b3f8d23648a58d38dfe2fd945c8459e9fdb62978a2bec0418c48/diff,workdir=/var/lib/docker/overlay2/ba5316850213b3f8d23648a58d38dfe2fd945c8459e9fdb62978a2bec0418c48/work
```
* Fetch the process id of the conatiner
```bash
docker inspect -f '{{.State.Pid}}' objective_fermat

27685
```
* Another way to do this is by using findmnt command
```bash
findmnt -N 27685

TARGET                  SOURCE               FSTYPE  OPTIONS
/                       overlay              overlay rw,relatime,seclabel,lowerdir=/var/lib/docker/overlay2/l/EYESVL37XVDGZUJQGX
├─/proc                 proc                 proc    rw,nosuid,nodev,noexec,relatime
│ ├─/proc/bus           proc[/bus]           proc    ro,nosuid,nodev,noexec,relatime
│ ├─/proc/fs            proc[/fs]            proc    ro,nosuid,nodev,noexec,relatime
│ ├─/proc/irq           proc[/irq]           proc    ro,nosuid,nodev,noexec,relatime
│ ├─/proc/sys           proc[/sys]           proc    ro,nosuid,nodev,noexec,relatime
│ ├─/proc/sysrq-trigger proc[/sysrq-trigger] proc    ro,nosuid,nodev,noexec,relatime
│ ├─/proc/acpi          tmpfs                tmpfs   ro,relatime,seclabel
│ ├─/proc/kcore         tmpfs[/null]         tmpfs   rw,nosuid,seclabel,size=65536k,mode=755
│ ├─/proc/keys          tmpfs[/null]         tmpfs   rw,nosuid,seclabel,size=65536k,mode=755
│ ├─/proc/latency_stats tmpfs[/null]         tmpfs   rw,nosuid,seclabel,size=65536k,mode=755
│ ├─/proc/timer_list    tmpfs[/null]         tmpfs   rw,nosuid,seclabel,size=65536k,mode=755
│ └─/proc/scsi          tmpfs                tmpfs   ro,relatime,seclabel
├─/dev                  tmpfs                tmpfs   rw,nosuid,seclabel,size=65536k,mode=755
│ ├─/dev/pts            devpts               devpts  rw,nosuid,noexec,relatime,seclabel,gid=5,mode=620,ptmxmode=666
│ ├─/dev/mqueue         mqueue               mqueue  rw,nosuid,nodev,noexec,relatime,seclabel
│ └─/dev/shm            shm                  tmpfs   rw,nosuid,nodev,noexec,relatime,seclabel,size=65536k
├─/sys                  sysfs                sysfs   ro,nosuid,nodev,noexec,relatime,seclabel
│ ├─/sys/firmware       tmpfs                tmpfs   ro,relatime,seclabel
│ └─/sys/fs/cgroup      cgroup[/system.slice/docker-47778c7349bcad5c2f14025a7d46ea94edc077e7f3a81c7ad3fb26f763206bb0.scope]
│                                            cgroup2 ro,nosuid,nodev,noexec,relatime,seclabel,nsdelegate,memory_recursiveprot
├─/etc/resolv.conf      /dev/vda1[/var/lib/docker/containers/47778c7349bcad5c2f14025a7d46ea94edc077e7f3a81c7ad3fb26f763206bb0/resolv.conf]
│                                            xfs     rw,noatime,seclabel,attr2,inode64,logbufs=8,logbsize=32k,sunit=1024,swidth=
├─/etc/hostname         /dev/vda1[/var/lib/docker/containers/47778c7349bcad5c2f14025a7d46ea94edc077e7f3a81c7ad3fb26f763206bb0/hostname]
│                                            xfs     rw,noatime,seclabel,attr2,inode64,logbufs=8,logbsize=32k,sunit=1024,swidth=
└─/etc/hosts            /dev/vda1[/var/lib/docker/containers/47778c7349bcad5c2f14025a7d46ea94edc077e7f3a81c7ad3fb26f763206bb0/hosts]
                                             xfs     rw,noatime,seclabel,attr2,inode64,logbufs=8,logbsize=32k,sunit=1024,swidth=
```
* You can also use Linux tooling (nsenter) to interact with the namespaces created by Docker. This is a useful technique when troubleshooting containers or investigating possibly malicious activity occurring in a container
``` bash
sudo nsenter --target 27685 --mount ls /

bin   dev		   docker-entrypoint.sh  home  lib64  mnt  proc  run   srv  tmp  var
boot  docker-entrypoint.d  etc			 lib   media  opt  root  sbin  sys  usr
```

#### PID namespace
* The PID namespace allows a process to have an isolated view of other processes running on the host. Containers use PID namespaces to ensure that they can only see and affect processes that are part of the contained application
* Note: Multiple containers may also share the same PID namespace. 
* This can be helpful for troubleshooting, as you can create a diagnostics container in the same namespace as an application container, and use it to run troubleshooting tools on the main application process.
* Start a container using busybox image and run top in the background 
```bash
 docker run --name busybox1 -d busybox top
```
* Use docker inspect to get the pid
```bash
docker inspect -f '{{.State.Pid}}' busybox1

2565
```
* Use nsenter to show the list of processes running inside a container.
```bash
sudo nsenter --target 2565 -m -p ps -ef

PID   USER     TIME  COMMAND
    1 root      0:00 top
    8 root      0:00 ps -ef
```
* you can observe that top process is running
* Another way to demonstrate the PID namespace is to use Linux’s unshare utility to run a program in a new set of namespaces
```bash
sudo unshare --pid --fork --mount-proc /bin/bash

[root@container-sec ec2-user]# 
```
* This provide us with a bash shell in a new PID namespace
* Run the ps -ef in the new bash shell
```bash
[root@container-sec ec2-user]# ps -ef
UID          PID    PPID  C STIME TTY          TIME CMD
root           1       0  0 10:41 pts/1    00:00:00 /bin/bash
root          18       1  0 10:45 pts/1    00:00:00 ps -ef
```
* Run exit to come out of the shell
```bash
exit
```
* When running containers, it can also be helpful to use PID namespaces to see the processes running in another container
* Note: The --pid switch on docker run allows us to start a container for debugging purposes in the process namespace of another container
* To understand this, start a web server container
```bash
docker run -d --name webserver nginx
```
* Start a debugging container using --pid option
```bash
docker run -it --name debug --pid=container:webserver raesene/alpine-containertools /bin/bash
```
* Run the ps -ef command in the bash shell of the debug container
```bash
bash-5.1# ps -ef
PID   USER     TIME  COMMAND
    1 root      0:00 nginx: master process nginx -g daemon off;
   29 101       0:00 nginx: worker process
   30 101       0:00 nginx: worker process
   31 root      0:00 /bin/bash
   37 root      0:00 ps -ef
```
```bash
exit
```
Note: Sharing the process namespace across containers is also possible in Kubernetes clusters, where it can be useful for debugging issues. If you want to share namespaces across a pod, it requires an option to be passed when the workload you want to debug is started. Specifically, you need to include shareProcessNamespace: true in your pod specification. More info https://kubernetes.io/docs/tasks/configure-pod-container/share-process-namespace/

#### Network namespace
* The net namespace is responsible for providing a process's network environment (interfaces, routing, etc.). 
* It is very useful for ensuring that contained processes can bind the ports they need without interfering with each other, and for verifying that traffic can be directed to specific applications.
* it’s possible to interact with the network namespace by using standard Linux tools like nsenter
* Get the container PID
```bash
docker inspect -f '{{.State.Pid}}' webserver

2758
```
* Use the -n switch on nsenter to enter the network namespace
```bash
sudo nsenter --target 2758 -n ip addr

1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
6: eth0@if7: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether 02:42:ac:11:00:03 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 172.17.0.3/16 brd 172.17.255.255 scope global eth0
       valid_lft forever preferred_lft forever
```
* Note: An important point here is that the ip command we’re running is being sourced from the host VM and doesn’t have to exist inside the container. This makes it a useful technique for troubleshooting networking issues in locked down containers that don’t have a lot of utilities installed in them.
* Note: It is possible to use Docker to share network namespaces, similarly to getting containers to share the PID namespace. 
* Launch a debugging container, with tools like tcpdump installed, and connect it to the network of the running container.
``` bash
docker run -it --name=debug-network --network=container:webserver raesene/alpine-containertools /bin/bash
```
* Run netstat -tunap to see listening ports, and it will show the web server running on port 80 from the other container
```bash
bash-5.1# netstat -tunap
Active Internet connections (servers and established)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name    
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      -
tcp        0      0 :::80                   :::*                    LISTEN      -
```
* Note: In Kubernetes environments, network namespace sharing will typically be in place for all containers in a single pod. Although you cannot launch a debugging container in an existing pod, you can use the new ephemeral containers feature to dynamically add a container to the pod’s network namespace. https://kubernetes.io/docs/tasks/debug/debug-application/debug-running-pod/#ephemeral-container
```bash
kubectl run webserver --image=nginx
kubectl debug -it --image raesene/alpine-containertools /bin/bash
netstat -tunap
kubectl debug -it --image raesene/alpine-containertools --target webserver /bin/bash
```

#### Cgroup namespace
* cgroups are designed to help control a process's resource usage on a Linux system. In containerization, they’re used to reduce the risk of “noisy neighbors” 
* First we’ll start a container and look at the number of entries in /sys/fs/cgroup
``` bash
docker run ubuntu:22.04 ls -R /sys/fs/cgroup | wc -l

Unable to find image 'ubuntu:22.04' locally
22.04: Pulling from library/ubuntu
a8b1c5f80c2d: Pull complete 
Digest: sha256:a6d2b38300ce017add71440577d5b0a90460d0e57fd7aec21dd0d1b0761bbfb2
Status: Downloaded newer image for ubuntu:22.04
70
```
* Note: Traditionally, cgroups assigned to processes were not namespaced, so there was some risk that information about processes would leak from one container to another. This led to the introduction of the cgroup namespace, which gives containers their own isolated cgroups
* But if we create another container that uses the host's cgroup namespace, we can see a lot more information available in that filesystem:

```bash
docker run --cgroupns=host ubuntu:22.04 ls -R /sys/fs/cgroup | wc -l

3320
```
* Looking in the /sys/fs/cgroup/system.slice/ directory of a container with access to the host's cgroup namespace, we can see that it contains information about system services running on the host. This is an example of the type of information leakage that is mitigated by using an isolated cgroup namespace.
```bash
docker run --cgroupns=host ubuntu:22.04 ls /sys/fs/cgroup/system.slice/

atd.service
auditd.service
boot-efi.mount
cgroup.controllers
cgroup.events
cgroup.freeze
cgroup.kill
cgroup.max.depth
cgroup.max.descendants
cgroup.pressure
cgroup.procs
cgroup.stat
cgroup.subtree_control
cgroup.threads
cgroup.type
chronyd.service
cloud-final.service
cloud-init-local.service
containerd.service
cpu.idle
cpu.max
cpu.max.burst
cpu.pressure
cpu.stat
cpu.weight
cpu.weight.nice
cpuset.cpus
cpuset.cpus.effective
cpuset.cpus.partition
cpuset.mems
cpuset.mems.effective
dbus-broker.service
docker-3530bf62ff56e8dec92102e39e1a9d11a9a0893a3b762fb5a92637757b92bb04.scope
docker.service
docker.socket
gssproxy.service
hugetlb.1GB.current
hugetlb.1GB.events
hugetlb.1GB.events.local
hugetlb.1GB.max
hugetlb.1GB.numa_stat
hugetlb.1GB.rsvd.current
hugetlb.1GB.rsvd.max
hugetlb.2MB.current
hugetlb.2MB.events
hugetlb.2MB.events.local
hugetlb.2MB.max
hugetlb.2MB.numa_stat
hugetlb.2MB.rsvd.current
hugetlb.2MB.rsvd.max
io.bfq.weight
io.max
io.pressure
io.stat
io.weight
irqbalance.service
libstoragemgmt.service
memory.current
memory.events
memory.events.local
memory.high
memory.low
memory.max
memory.min
memory.numa_stat
memory.oom.group
memory.peak
memory.pressure
memory.reclaim
memory.stat
memory.swap.current
memory.swap.events
memory.swap.high
memory.swap.max
memory.zswap.current
memory.zswap.max
misc.current
misc.events
misc.max
pids.current
pids.events
pids.max
pids.peak
rngd.service
sshd.service
sysroot.mount
system-getty.slice
system-modprobe.slice
system-serial\x2dgetty.slice
system-sshd\x2dkeygen.slice
system-systemd\x2dfsck.slice
systemd-homed.service
systemd-journald.service
systemd-logind.service
systemd-networkd.service
systemd-resolved.service
systemd-udevd.service
systemd-userdbd.service
tmp.mount
var-lib-nfs-rpc_pipefs.mount
```

#### UTS namespace
* Another less commonly used namespace with a relatively specific purpose: setting the hostname used by a process
* Note: Linux container runtimes activate this namespace by default, which is why containers have different hostnames than their underlying VMs.
```bash
docker run -it ubuntu:22.04 /bin/bash

root@1824bde1f41d:/# exit
exit
```
* We can share the host’s UTS namespace using the --uts=host flag
```bash
[ec2-user@container-sec ~]$ docker run --uts=host -it ubuntu:22.04 /bin/bash
root@container-sec:/# 
```
* Note: Linux namespaces are a foundational part of how container runtimes like Docker work. We've seen how they can provide fine-grained isolation of a container’s view of the host’s resources in a number of ways