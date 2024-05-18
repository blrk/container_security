### Seccomp a "last line of defence"
Objective: Evaluate how seccomp filters are used as a "last line of defense" by container runtimes.

#### Seccomp filters in Docker containers
* As part of the default set of restrictions that Docker puts in place, a seccomp filter is applied to any new container.
* This filter can catch cases where an action in a container would be allowed by the other layers of protection (e.g., capabilities)
* Create a docker container and try the unshare command
* Note: CVE-2022-0185, which uses the unshare syscall to exploit a vulnerability.
```bash
docker run -it ubuntu:22.04 /bin/bash

root@8bc9597c96b0:/# unshare
unshare: unshare failed: Operation not permitted
```
* Exit from the container
* Next, we can start a container with the seccomp filter explicitly disabled
```bash
docker run --security-opt seccomp=unconfined -it ubuntu:22.04 /bin/bash

root@b1bbb0cadbb1:/# unshare
# 
```
* Run exit twice to come out of the container
#### Creating custom seccomp filters
* Although the Docker default seccomp profile provides a good level of isolation, there are some scenarios that call for different degrees of restrictions
* Get the secomp profile that blocks the chmod system calls
```bash
wget https://raw.githubusercontent.com/blrk/container_security/main/files/no-chmod.json
```
* Start a container using this profile
```bash
docker run -it --security-opt seccomp=no-chmod.json ubuntu:22.04 /bin/bash

root@d7980f8b9027:/# chmod 744 /etc/passwd
chmod: changing permissions of '/etc/passwd': Operation not permitted
```
* exit from the container
* Track all the systemcalls from a container using strace
```bash
strace -fvttTyy -s 256 -o strace.txt docker run -it ubuntu:22.04 ls / >/dev/null
```
* View the logs
```bash
cat  strace.txt | cut -f 4 -d " " | less
```