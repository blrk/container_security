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
