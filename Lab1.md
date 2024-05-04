### Exploring containers as processes
* Objective: Understand containers are processes, use Linux tools to interact with containers, and explore what this means for securing container environments

* Check if there are any active processes named nginx
``` bash
ps -fC nginx
```
* This should return an empty list, as we don’t have any NGINX web servers running at the moment
* Now, let's start a Docker container by using the nginx image from Docker Hub
``` bash
docker run -d nginx:1.23.1
```
* Once again, run the ps command
``` bash
ps -fC nginx

[ec2-user@container-sec ~]$ ps -fC nginx
UID          PID    PPID  C STIME TTY          TIME CMD
root       26294   26271  0 06:44 ?        00:00:00 nginx: master process nginx -g daemon off;
101        26338   26294  0 06:44 ?        00:00:00 nginx: worker process
101        26339   26294  0 06:44 ?        00:00:00 nginx: worker process
```
* As far as our Linux machine is concerned, someone just ran NGINX on the host
* check for running containers 
``` bash
docker ps
```
* Alternatively, we could use Linux process tools to determine if the web server is running as a container. The --forest option on ps (for example, ps -ef --forest) lets us see a hierarchy of processes. In this case, our NGINX processes have a parent process of containerd-shim-runc-v2. 

``` bash
ps -ef --forest

root       26271       1  0 06:44 ?        00:00:00 /usr/bin/containerd-shim-runc-v2 -namespace moby -id b0e14a851eb861d58cf73063b5
root       26294   26271  0 06:44 ?        00:00:00  \_ nginx: master process nginx -g daemon off;
101        26338   26294  0 06:44 ?        00:00:00      \_ nginx: worker process
101        26339   26294  0 06:44 ?        00:00:00      \_ nginx: worker process
root       26372       1  2 06:55 ?        00:00:00 /usr/lib/systemd/systemd-hostnamed
```
* Note: You should see a shim process for each container running on the host. This shim process is part of containerd and is used by Docker to manage contained processes. The goal of this shim process is to allow containerd or the Docker daemon to be restarted without having to restart all the containers running on the host.

#### Interacting with a container as a process
* Being able to interact with them as processes is useful for both troubleshooting their operation and investigating changes in running containers (for example, in a forensic investigation).
* First is that we can use the /proc filesystem to get more information about our running containers
* Note: The /proc filesystem in Linux is a virtual or pseudo filesystem. It doesn’t contain real files—instead, it is populated with information about the running system. 
* 
``` bash
ls /proc

1     1109  1222  2     2062  2288   26352  28   40         bus            fs           latency_stats  schedstat      uptime
10    1113  1223  20    2099  23     26364  29   41         cgroups        interrupts   loadavg        scsi           version
1008  1119  1267  201   21    234    26367  3    42         cmdline        iomem        locks          self           vmallocinfo
1031  1121  13    2023  2103  244    26368  30   43         consoles       ioports      mdstat         slabinfo       vmstat
1039  1125  1311  2025  2160  245    26396  32   5          cpuinfo        irq          meminfo        softirqs       zoneinfo
1051  1126  14    203   2162  25943  26400  33   6          crypto         kallsyms     misc           stat
1063  12    15    2033  2164  25955  26403  34   65         devices        kcore        modules        swaps
1085  1216  16    2034  223   26     26404  35   70         diskstats      key-users    mounts         sys
1099  1217  18    2035  224   26171  26405  36   72         dma            keys         mtrr           sysrq-trigger
11    1218  19    2037  2268  26271  26406  360  74         driver         kmsg         net            sysvipc
1101  1219  1942  2038  2276  26294  26407  37   8          dynamic_debug  kpagecgroup  pagetypeinfo   thread-self
1103  1220  1943  2039  2281  26338  26409  39   acpi       execdomains    kpagecount   partitions     timer_list
1107  1221  199   2043  2286  26339  27     4    buddyinfo  filesystems    kpageflags   pressure       tty
```
* Let’s look at some information about the NGINX container we started up earlier. On the test system we're using, we can see that the nginx process ID is 26294

* It can also be helpful to use Linux tooling to work with containers that have been hardened to remove tools such as file editors or process monitors. 
* Note: Hardening container images is a common security recommendation, but it does make debugging trickier. 
* You can edit files inside the container by accessing the container’s root filesystem from the /proc directory on the host. 
* Navigating to /proc/[PID]/root will give you the directory listing of the contained process that has that PID

``` bash
sudo ls /proc/26294/root
bin   dev		   docker-entrypoint.sh  home  lib64  mnt  proc  run   srv  tmp  var
boot  docker-entrypoint.d  etc			 lib   media  opt  root  sbin  sys  usr
```
* Now, if we use touch to add a file to this directory, we can confirm that it’s been added by using docker exec to list files on the container. 
* This technique can be used to do things like edit configuration files in containers from the host.

``` bash
sudo touch /proc/26294/root/newfile_rk
```
``` bash
docker exec b0e ls /

bin
boot
dev
docker-entrypoint.d
docker-entrypoint.sh
etc
home
lib
lib64
media
mnt
newfile_rk
opt
proc
root
run
sbin
srv
sys
tmp
usr
var
```
* And here's another benefit of containers being processes: we can use host-level tools to kill those processes without needing to use container tools. 
* Note: This isn’t something that’s a good idea for general use, as it could interact oddly with settings like Docker’s restart policy, but there may be times when it’s necessary.

* Try to kill the container using process id
```bash
sudo kill 26294
```
* List the containers
```bash
 docker ps
CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES
```
* Security Pointers: This has some interesting consequences for security. First, we need to account for the fact that anyone who has access to the underlying host can use process lists to see information about running containers—even if they can’t directly access tools like Docker.
* A user with access to the underlying host can read the contents of the environ file inside a process's area in /proc to see that information
* Start a postgres container
``` bash
docker run --name db -e POSTGRES_PASSWORD=mysecretpassword -d postgres
```
* Identify the process id
```bash 
ps -fC postgres

UID          PID    PPID  C STIME TTY          TIME CMD
systemd+   26792   26771  0 07:33 ?        00:00:00 postgres
systemd+   26869   26792  0 07:33 ?        00:00:00 postgres: checkpointer 
systemd+   26870   26792  0 07:33 ?        00:00:00 postgres: background writer 
systemd+   26872   26792  0 07:33 ?        00:00:00 postgres: walwriter 
systemd+   26873   26792  0 07:33 ?        00:00:00 postgres: autovacuum launcher 
systemd+   26874   26792  0 07:33 ?        00:00:00 postgres: logical replication launcher 
```
* List the environment varible from process area
``` bash
sudo cat /proc/26792/environ 

HOSTNAME=48856ed902f6POSTGRES_PASSWORD=mysecretpasswordPWD=/HOME=/var/lib/postgresqlLANG=en_US.utf8GOSU_VERSION=1.17POSTGRES_INITDB_ARGS=PG_MAJOR=16PG_VERSION=16.2-1.pgdg120+2SHLVL=0POSTGRES_USER=postgresPGDATA=/var/lib/postgresql/dataPATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/lib/postgresql/16/binPOSTGRES_DB=postgres
```