### Container Capabilities
* Start a ubuntu container
```bash
docker run -it ubuntu:22.04 /bin/bash
```
* Try changing the time with date
```bash
root@16e3ebf37d2f:/# date +%T -s "09:09:09"

date: cannot set date: Operation not permitted
09:09:09
```
* exit from the container
```bash
exit
```
* Note: root user who would usually be able to set the time on a host, there must be some other feature that's preventing this from working

#### Linux capabilities
* Capabilities split up the monolithic root privilege into 41 (at the time of preparing this manual) privileges that can be individually granted to processes or files
* Install libcap-ng-utils to use the pscap, that list the capabilities have been granted to processes on our host
```bash
sudo yum install libcap-ng-utils -y
```
* run the pscap in the host
```bash
pscap 

ppid  pid   name        command             capabilities
1     1267  root        systemd-journal     chown, dac_override, dac_read_search, fowner, setgid, setuid, sys_ptrace, sys_admin, audit_control, mac_override, syslog, audit_read +
1     1313  root        systemd-udevd       chown, dac_override, dac_read_search, fowner, fsetid, kill, setgid, setuid, setpcap, linux_immutable, net_bind_service, net_broadcast, net_admin, net_raw, ipc_lock, ipc_owner, sys_module, sys_rawio, sys_chroot, sys_ptrace, sys_pacct, sys_admin, sys_boot, sys_nice, sys_resource, sys_tty_config, mknod, lease, audit_write, audit_control, setfcap, mac_override, mac_admin, syslog, block_suspend, audit_read, perfmon, bpf, checkpoint_restore +
1     1945  systemd-resolve  systemd-resolve     net_raw @ +
1     1946  root        auditd              chown, dac_override, dac_read_search, fowner, fsetid, kill, setgid, setuid, setpcap, linux_immutable, net_bind_service, net_broadcast, net_admin, net_raw, ipc_lock, ipc_owner, sys_rawio, sys_chroot, sys_ptrace, sys_pacct, sys_admin, sys_boot, sys_nice, sys_resource, sys_time, sys_tty_config, mknod, lease, audit_write, audit_control, setfcap, mac_override, mac_admin, syslog, wake_alarm, block_suspend, audit_read, perfmon, bpf, checkpoint_restore +
2028  2029  dbus        dbus-broker         audit_write +
1     2034  root        rngd                full +
1     2036  root        systemd-homed       chown, dac_override, dac_read_search, fowner, fsetid, setgid, setuid, setpcap, sys_admin, sys_resource, setfcap +
1     2037  root        systemd-logind      chown, dac_override, dac_read_search, fowner, linux_immutable, sys_admin, sys_tty_config, audit_control, mac_admin +
1     2067  chrony      chronyd             net_bind_service, sys_time +
1     2098  systemd-network  systemd-network     net_bind_service, net_broadcast, net_admin, net_raw @ +
1     2104  root        containerd          full +
1     2105  root        gssproxy            full +
1     2161  root        sshd                full +
1     2164  root        dockerd             full +
1     2165  root        atd                 full +
1     2166  root        agetty              full +
1     2167  root        agetty              full +
2161  2382  root        sshd                full +
1     2385  root        systemd-userdbd     dac_read_search, sys_resource +
2385  2716  root        systemd-userwor     dac_read_search, sys_resource +
2385  2717  root        systemd-userwor     dac_read_search, sys_resource +
2385  2718  root        systemd-userwor     dac_read_search, sys_resource +
```
* Note: There are few processes are running with full capabilities, others have been given limited capabilities
#### Capabilities and containers
* Let’s take a look at how capabilities are used in containers. 
* Start by running a container 
```bash
docker run -d nginx 
```
* Use pscap again to see what has changed
```bash
pscap | grep -i nginx

2793  2817  root        nginx               chown, dac_override, fowner, fsetid, kill, setgid, setuid, setpcap, net_bind_service, net_raw, sys_chroot, mknod, audit_write, setfcap +
```
* The above screenshot shows the set of capabilities that Docker grants to containers by default. This set was designed so that most workloads would be able to run inside containers successfully.
* Alternatively, its possible to check which capabilities a container has from the inside of the container using amicontained
```bash
docker run -it raesene/alpine-containertools /bin/bash

bash-5.1# amicontained 
Container Runtime: not-found
Has Namespaces:
	pid: true
	user: false
AppArmor Profile: system_u:system_r:unconfined_service_t:s0
Capabilities:
	BOUNDING -> chown dac_override fowner fsetid kill setgid setuid setpcap net_bind_service net_raw sys_chroot mknod audit_write setfcap
Seccomp: filtering
Blocked Syscalls (54):
	MSGRCV SYSLOG SETSID USELIB USTAT SYSFS VHANGUP PIVOT_ROOT _SYSCTL ACCT SETTIMEOFDAY MOUNT UMOUNT2 SWAPON SWAPOFF REBOOT SETHOSTNAME SETDOMAINNAME IOPL IOPERM CREATE_MODULE INIT_MODULE DELETE_MODULE GET_KERNEL_SYMS QUERY_MODULE QUOTACTL NFSSERVCTL GETPMSG PUTPMSG AFS_SYSCALL TUXCALL SECURITY LOOKUP_DCOOKIE CLOCK_SETTIME VSERVER MBIND SET_MEMPOLICY GET_MEMPOLICY KEXEC_LOAD ADD_KEY REQUEST_KEY KEYCTL MIGRATE_PAGES UNSHARE MOVE_PAGES PERF_EVENT_OPEN FANOTIFY_INIT OPEN_BY_HANDLE_AT SETNS KCMP FINIT_MODULE KEXEC_FILE_LOAD BPF USERFAULTFD
Looking for Docker.sock
```
* exit from the container
```bash
exit
```
* Note: It’s possible to either increase or decrease a container's rights by using Docker's cap-add and cap-drop flags to add and drop capabilities

#### Minimizing a container's set of capabilities
* List the capabilities of a running container
```bash
docker run -d fedora sleep 5 >/dev/null; pscap | grep sleep

3334  3358  root        sleep               chown, dac_override, fowner, fsetid, kill, setgid, setuid, setpcap, net_bind_service, net_raw, sys_chroot, mknod, audit_write, setfcap +
```
* You have to drop setfcap, audit_write, and mknod, you could use --cap-drop=setfcap --cap-drop=audit_write --cap-drop=mknod
```bash
docker run -d --cap-drop=setfcap --cap-drop=audit_write --cap-drop=mknod fedora sleep 5 > /dev/null; pscap | grep sleep

3451  3472  root        sleep               chown, dac_override, fowner, fsetid, kill, setgid, setuid, setpcap, net_bind_service, net_raw, sys_chroot +
```
*  If you know your container only needs setuid and setgid, you can drop all capabilities and just add setgid and setuid back in
```bash
docker run -d --cap-drop=all --cap-add=setuid --cap-add=setgid fedora sleep 5 > /dev/null; pscap | grep sleep

3567  3590  root        sleep               setgid, setuid +
```
#### The most dangerous capability: SYS_ADMIN
* Note: One basic way to test this out on your container is to drop all capabilities at once (i.e., by running your application container with the cap-drop=ALL) and then monitor for errors as it completes a run of its test suite. This should help you confirm if your container actually needs any specific capabilities to do its work.

#### Creating Semi-Privileged Environments
* Allow us to create environments that allow certain capabilities to be used

#### cgroups v1 and v2

* Control group v2 provides management benefits over the original implementation and is required for certain container features, such as rootless containers.
* Note: Control group v2 was initially introduced in version 4.5 of the Linux kernel in 2016
* Detect the version of cgroup use the following command
```bash
mount | grep cgroup
cgroup2 on /sys/fs/cgroup type cgroup2 (rw,nosuid,nodev,noexec,relatime,seclabel,nsdelegate,memory_recursiveprot)
```
* To view the cgroup information in an organised way, use systemd-cgls
```bash
systemd-cgls

Control group /:
-.slice
├─user.slice (#1141)
│ → user.invocation_id: cc2440a5544942e7a9be114200506931
│ └─user-1000.slice (#4424)
│   → user.invocation_id: 9a34275a73674570a1e37ec9c321413e
│   ├─user@1000.service … (#4512)
│   │ → user.delegate: 1
│   │ → user.invocation_id: fbb954bf7c4b4526b0972ae48b438bf7
│   │ └─init.scope (#4556)
│   │   ├─2390 /usr/lib/systemd/systemd --user
│   │   └─2392 (sd-pam)
│   └─session-1.scope (#4722)
│     ├─2382 sshd: ec2-user [priv]
│     ├─2399 sshd: ec2-user@pts/0
│     ├─2400 -bash
│     ├─4861 systemd-cgls
│     └─4862 less
├─init.scope (#23)
│ └─1 /usr/lib/systemd/systemd --switched-root --system --deserialize=32
└─system.slice (#62)
  ├─rngd.service (#3233)
  │ → user.invocation_id: 56032927d00d41b0acf9eeceff5de7c0
  │ └─2034 /usr/sbin/rngd -f -x pkcs11 -x nist
  ├─irqbalance.service (#3127)

```
* The above screenshot is trimmed
#### Using cgroups to limit resources
* Now that we have an understanding of how to view cgroup information, the next step is to explore how we can use cgroups to restrict the resources available to processes, which can help alleviate denial-of-service risks. 
* To demonstrate this, we will employ the stress tool to simulate an attacker or misbehaving application consuming all of the CPU on our host.
* Inside a Docker container, we can utilize the command stress -c 2, which will start two processes that consume a total of 2 cores of CPU. Then, by executing the top command in another window, we can verify the effect on the host's CPU.
* In terminal-1 
```bash
docker run -it --rm blrk/stress bash

bash-5.2# stress -c 2
stress: info: [7] dispatching hogs: 2 cpu, 0 io, 0 vm, 0 hdd
```
* In terminal-2
```bash
top - 16:29:10 up 12:07,  2 users,  load average: 1.94, 1.02, 0.41
Tasks: 128 total,   3 running, 123 sleeping,   2 stopped,   0 zombie
%Cpu(s):100.0 us,  0.0 sy,  0.0 ni,  0.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
MiB Mem :   3907.2 total,   2140.1 free,    391.6 used,   1375.5 buff/cache
MiB Swap:      0.0 total,      0.0 free,      0.0 used.   3228.5 avail Mem 

    PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND                                                      
   7237 root      20   0    3512    104      0 R 100.0   0.0   3:24.98 stress                                                       
   7238 root      20   0    3512    104      0 R  99.7   0.0   3:24.91 stress  
```
* exit from the container
* Note: Docker offers various options for limiting the amount of CPU time the container can utilize, but the simplest is the --cpus flag, which allows you to specify a decimal number of CPUs that can be utilized. Under the covers, Docker leverages cgroups to enforce this limit
* In terminal-1
```bash
 docker run -it --rm --cpus 0.5 blrk/stress bash
bash-5.2# stress -c 2
stress: info: [7] dispatching hogs: 2 cpu, 0 io, 0 vm, 0 hdd
```
* In terminal-2
```bash
top - 16:34:46 up 12:12,  2 users,  load average: 0.35, 0.71, 0.49
Tasks: 129 total,   3 running, 124 sleeping,   2 stopped,   0 zombie
%Cpu(s): 25.2 us,  0.2 sy,  0.0 ni, 74.6 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
MiB Mem :   3907.2 total,   2142.5 free,    389.1 used,   1375.6 buff/cache
MiB Swap:      0.0 total,      0.0 free,      0.0 used.   3231.0 avail Mem 

    PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND                                                      
   7401 root      20   0    3512    104      0 R  25.8   0.0   0:23.37 stress                                                       
   7400 root      20   0    3512    104      0 R  24.2   0.0   0:23.48 stress 
```
* exit from the container
#### Using cgroups to defeat fork bombs
* A common denial-of-service attack on Linux systems is known as a fork bomb, which occurs when an attacker generates a very large number of processes, ultimately depleting the system's resources. By default, containers are not restricted in terms of how many new processes they can generate, which means that any process can create a fork bomb
* Cgroups have the ability to restrict the number of processes that can be spawned, which effectively safeguards the host from a fork bomb attack. We can demonstrate this by using the --pids-limit flag

```bash 
docker run -it --pids-limit 10 ubuntu:22.04 /bin/bash
root@7914bfcbad1f:/# :(){ :|: & };:

[1] 10
root@7914bfcbad1f:/# bash: fork: retry: Resource temporarily unavailable
bash: fork: retry: Resource temporarily unavailable
bash: fork: retry: Resource temporarily unavailable
bash: fork: retry: Resource temporarily unavailable
```
* Note: Very quickly, the container reaches the limit of 10 processes, and errors are displayed. However, the underlying host will remain responsive, preventing the denial-of-service attack

#### Using cgroups to control device access
* Another security-related aspect of cgroups is that they can be used to control access to devices. Containers provide access to a range of devices on the host machine, as detailed in runc's allowed devices list, and it is possible to utilize Docker's functionality (which uses cgroups) to add other devices to that list. This allows you to give specific containers access to hardware (such as an audio device).

```bash
docker run -d --rm --device /dev/dm-0 --name webdevice nginx
```



