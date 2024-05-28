### Remove setuid/setgid Binaries
Objective: This exercise covers permissions that can be set for setuid and setgid binaries and best practices for managing them.
* SETUID/SETGID Overview: There are two special permissions that can be set on executable files: Set User ID (setuid) and Set Group ID (sgid). These permissions allow the file being executed to be executed with the privileges of the owner or the group. For example, if a file was owned by the root user and has the setuid bit set, no matter who executed the file it would always run with root user privileges.
* Find a list of binaries: To get a list of binaries with special permissions in a container image, the following syntax can be used find / -perm +6000 -type f -exec ls -ld {} \;
```bash
sudo podman run debian:jessie find / -perm +6000 -type f -exec ls -ld {} \; 2> /dev/null

-rwsr-xr-x. 1 root root 40000 Mar 29  2015 /bin/mount
-rwsr-xr-x. 1 root root 44104 Nov  8  2014 /bin/ping
-rwsr-xr-x. 1 root root 44552 Nov  8  2014 /bin/ping6
-rwsr-xr-x. 1 root root 40168 May 17  2017 /bin/su
-rwsr-xr-x. 1 root root 27416 Mar 29  2015 /bin/umount
-rwxr-sr-x. 1 root shadow 35408 May 27  2017 /sbin/unix_chkpwd
-rwxr-sr-x. 1 root shadow 62272 May 17  2017 /usr/bin/chage
-rwsr-xr-x. 1 root root 53616 May 17  2017 /usr/bin/chfn
-rwsr-xr-x. 1 root root 44464 May 17  2017 /usr/bin/chsh
-rwxr-sr-x. 1 root shadow 22744 May 17  2017 /usr/bin/expiry
-rwsr-xr-x. 1 root root 75376 May 17  2017 /usr/bin/gpasswd
-rwsr-xr-x. 1 root root 39912 May 17  2017 /usr/bin/newgrp
-rwsr-xr-x. 1 root root 54192 May 17  2017 /usr/bin/passwd
-rwxr-sr-x. 1 root tty 27232 Mar 29  2015 /usr/bin/wall
```
* "Defang" the binaries: You can then “defang” the binaries with chmod a-s to remove the suid bit. 
* Create a directory and navigate into that 
```bash
mkdir ~/defanged-debian; cd ~/defanged-debian
```
* Copy the Dockerfile text below and paste it. 
```bash
vi Dockerfile

FROM debian:jessie
RUN find / -xdev -perm +6000 -type f -exec chmod a-s {} \; || true
```
* The || true allows you to ignore any errors from find. The setuid and setgid binaries run with the privileges of the owner rather than the user. These are normally used to allow users to temporarily run with escalated privileges required to execute a given task, such as setting a password.
* Build the Image
```bash
docker build -t defanged-debian .
```
* Test the image.
```bash
docker run --rm defanged-debian \
  find / -perm +6000 -type f -exec ls -ld {} \; 2> /dev/null | wc

      0       0       0
```
* Run the command on your host
```bash
sudo find / -perm /6000 -type f -exec ls -ld {} \; 2> /dev/null | wc

    237    2133   33398
```
* It’s more likely that your Dockerfile will rely on a setuid/setgid binary than your application. Therefore, you can always perform this step near the end, after any such calls and before changing the user (removing setuid binaries is pointless if the application runs with root privileges).