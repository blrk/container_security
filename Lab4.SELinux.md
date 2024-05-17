### SELinux
* Objective: How SELinux can provide additional restrictions beyond the other layers of isolation 
#### Understanding SELinux
* SELinux labels Linux resources, such as files and ports, and restricts access to them based on each resource's labels and the properties of the process trying to access the resource.
* Run the following command in the terminal to see, how SELinux is configured in the system
```bash
sestatus

SELinux status:                 enabled
SELinuxfs mount:                /sys/fs/selinux
SELinux root directory:         /etc/selinux
Loaded policy name:             targeted
Current mode:                   permissive
Mode from config file:          permissive
Policy MLS status:              enabled
Policy deny_unknown status:     allowed
Memory protection checking:     actual (secure)
Max kernel policy version:      33
```
* This command returns key information about how SELinux is configured on this host.
* The first line indicates that SELinux is enabled. Loaded policy name tells us that we're running in targeted mode (which means that SELinux will be applied to specific processes chosen by the distribution provider (e.g., Red Hat) on the host), as opposed to mls mode, which is stricter and applies restrictions to every process. Typically, mls mode is not suitable for general purpose systems, due to the complexity of managing labeling and permissions on all processes.
* The next important line is: Current mode: enforcing. Here, the possible options are: enforcing, permissive, and disabled. Permissive mode is useful for setting up SELinux, as it will not block actions. Instead, it will log any denials that would have occurred if the system had been in enforcing mode.
* Verify that SELinux has a Docker context
```bash
ps -eZ | grep docker
system_u:system_r:unconfined_service_t:s0 2162 ? 00:00:02 dockerd
```
* This means SELinux does not manages the Docker daemon. Inspect the Docker daemon to see if SELinux is enabled by default
```bash
docker info | grep Security -A3
 Security Options:
  seccomp
   Profile: builtin
  cgroupns
```
* SELinux is not enabled by default. To fix it, enable SELinux to control and manage Docker by updating or creating the file /etc/docker/daemon.json
```bash
sudo vi /etc/docker/daemon.json
```
```bash
{
  "selinux-enabled": true
}
```
* Restart the docker service
```bash
sudo systemctl restart docker
```
* Verify SELinux is enabled
```bash
docker info | grep Security -A3
 Security Options:
  seccomp
   Profile: builtin
  selinux
```




* Install semanage 
```bash 
sudo yum install /usr/sbin/semanage -y 
```
* Explore more details about how SELinux is configured to handle standard user processes.
```bash
sudo semanage login -l

Login Name           SELinux User         MLS/MCS Range        Service

__default__          unconfined_u         s0-s0:c0.c1023       *
root                 unconfined_u         s0-s0:c0.c1023       *
```
* Note: From this output, we can see that SELinux considers ordinary users (denoted by __default__) and the root user to be unconfined, meaning that it won't apply restrictions to them.
* View the labels that SELinux uses
```bash
ps -efZ

system_u:system_r:sshd_t:s0-s0:c0.c1023 root 2160      1  0 06:44 ?        00:00:00 sshd: /usr/sbin/sshd -D [listener] 0 of 10-100 startups
system_u:system_r:unconfined_service_t:s0 root 2163    1  0 06:44 ?        00:00:04 /usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock --default-ulimit nofile=32768:65536
system_u:system_r:crond_t:s0-s0:c0.c1023 root 2164     1  0 06:44 ?        00:00:00 /usr/sbin/atd -f
system_u:system_r:getty_t:s0-s0:c0.c1023 root 2165     1  0 06:44 tty1     00:00:00 /sbin/agetty -o -p -- \u --noclear - linux
system_u:system_r:getty_t:s0-s0:c0.c1023 root 2166     1  0 06:44 ttyS0    00:00:00 /sbin/agetty -o -p -- \u --keep-baud 115200,57600,38400,9600 - vt220
system_u:system_r:sshd_t:s0-s0:c0.c1023 root 2418   2160  0 06:46 ?        00:00:00 sshd: ec2-user [priv]
system_u:system_r:systemd_userdbd_t:s0 root 2421       1  0 06:46 ?        00:00:00 /usr/lib/systemd/systemd-userdbd
unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023 ec2-user 2426 1  0 06:46 ? 00:00:00 /usr/lib/systemd/systemd --user
system_u:system_r:init_t:s0     ec2-user    2428    2426  0 06:46 ?        00:00:00 (sd-pam)
unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023 ec2-user 2435 2418  0 06:46 ? 00:00:00 sshd: ec2-user@pts/0
unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023 ec2-user 2436 2435  0 06:46 pts/0 00:00:00 -bash
```
* You can see that dockerd and containerd processes have the unconfined_service_t type applied to them using a label, and the ssh processe used sshd_t
* You an also see this information in file systems by using commands like ls -alZ
```bash
ls -alZ
total 36
drwx------. 5 ec2-user ec2-user unconfined_u:object_r:user_home_dir_t:s0  175 May  7 16:27 .
drwxr-xr-x. 3 root     root     system_u:object_r:home_root_t:s0           22 May  4 05:17 ..
-rw-------. 1 ec2-user ec2-user unconfined_u:object_r:user_home_t:s0     4670 May  7 16:46 .bash_history
-rw-r--r--. 1 ec2-user ec2-user unconfined_u:object_r:user_home_t:s0       18 Jan 28  2023 .bash_logout
-rw-r--r--. 1 ec2-user ec2-user unconfined_u:object_r:user_home_t:s0      141 Jan 28  2023 .bash_profile
-rw-r--r--. 1 ec2-user ec2-user unconfined_u:object_r:user_home_t:s0      492 Jan 28  2023 .bashrc
drwx------. 3 ec2-user ec2-user unconfined_u:object_r:config_home_t:s0     20 May  7 16:27 .config
drwx------. 3 ec2-user ec2-user unconfined_u:object_r:user_home_t:s0       82 May  7 16:23 .docker
-rw-------. 1 ec2-user ec2-user unconfined_u:object_r:user_home_t:s0       36 May  7 15:40 .lesshst
drwx------. 2 ec2-user ec2-user system_u:object_r:ssh_home_t:s0            29 May  4 05:17 .ssh
-rw-------. 1 ec2-user ec2-user unconfined_u:object_r:user_home_t:s0     5627 May  7 16:21 .viminfo
-rw-r--r--. 1 ec2-user ec2-user unconfined_u:object_r:user_home_t:s0       66 May  7 16:21 Dockerfile
```
#### Container SELinux policies
* When running Docker under Linux distributions (Example Redhat), a general SELinux policy will be applied to all new containers
* General profile has to make tradeoffs in the protection provided, as it applies the same policy to every container
* To see the effect of this policy, create a container named c1 that mounts the home directory into the container
```bash
docker run --rm -it --name c1 -v /home/ec2-user:/hosthome fedora /bin/bash
```
* Try to create a file inside the /hosthome directory inside the container
```bash

```