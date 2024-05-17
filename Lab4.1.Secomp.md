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
```bash
vi no_uring.json
```
```json
{
  "defaultAction": "SCMP_ACT_ERRNO",
  "architectures": [
    "SCMP_ARCH_X86_64",
    "SCMP_ARCH_X86",
    "SCMP_ARCH_X32"
  ],
  "syscalls": [
    "io_uring_enter",
    "io_uring_register",
    "io_uring_setup"
  ]
}
```
* using th
```bash
docker run -it --security-opt seccomp=no_uring.json ubuntu:22.04 /bin/bash

docker: Error response from daemon: Decoding seccomp profile failed: json: cannot unmarshal string into Go struct field Seccomp.syscalls of type seccomp.Syscall.
ERRO[0000] error waiting for container: context canceled 
```
#### 
* Download the custom secomp profile
```bash
wget  https://raw.githubusercontent.com/blrk/container_security/main/files/custom-seccomp.json
```
* The defaultAction is set to SCMP_ACT_ERRNO, which means that any system call not explicitly allowed will result in an error.
* The syscalls section specifies the system calls that are allowed (SCMP_ACT_ALLOW). This example allows chdir, fchown, fchownat, setuid, setgid, clone, fork, and vfork.
* Apply the Custom Seccomp Profile to a Docker Container
```bash

```