### Understanding container images & Software vulnerabilities in images
Objective: 
#### Root file system and image configuration
* Create directory and navigate into that
```bash
cd ~; mkdir images; cd images
```
* There are two parts in a container image. Root file system and configuration
* Download the alpine image
```bash
curl -o alpine.tar.gz https://dl-cdn.alpinelinux.org/alpine/v3.10/releases/x86_64/alpine-minirootfs-3.10.0-x86_64.tar.gz

  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 2647k  100 2647k    0     0  3089k      0 --:--:-- --:--:-- --:--:-- 3089k
```
* list the files in the directory. The alpine image tar is downloaded
```bash
ls

alpine.tar.gz
```
* Extract the tar file
```bash
tar -xvf alpine.tar.gz
```
* List the files
```bash
ls

alpine.tar.gz  bin  dev  etc  home  lib  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var
```
* Delete all the file form the image directory
```bash
cd ~; rm -Rf /home/<add your username here>/images/*
```
#### Understand OCI Standards
* Note: Skopeo is useful for manipulating and inspecting OCI images
* Start the skopeo container
```bash
docker run -itd --name skopeo blrk/skopeo:1.15.0 

1a6daa649b30cea8c6a577bca1271ad32328152f3ef4c3672acceb0fb5a55e24
```
* Get into the container
```bash
docker exec -it 1a6 bash
```
* Run the following command to copy a ontainer image from a Docker registry to an OCI (Open Container Initiative) image layout format
```bash
skopeo copy docker://alpine:latest oci:alpine:latest

Getting image source signatures
Copying blob 4abcf2066143 done   | 
Copying config bc4e4f7999 done   | 
Writing manifest to image destination
```
* List the files in alpine directory
```bash
ls alpine/

blobs  index.json  oci-layout
```
* OCI complaint run time like runc doesn't work directly with the image in this format.  It has to be unpacked into a runtime file system buldle. 
```bash
umoci unpack --image alpine:latest alpine-bundle
```
* List the files in alpine-bundle
```bash
 ls alpine-bundle/

config.json  rootfs  sha256_e58adedfeda064c24b8ed27ab606f59dd8488b6b623bacc9e105471ae2a1f76e.mtree  umoci.json
```
* As you can see, this bundle includes a rootfs directory with the contents of Alpine Linux distro and config.json which includes the runtime settings
* View the config.json 
```bash
cat alpine-bundle/config.json | jq

{
  "ociVersion": "1.0.2",
  "process": {
    "terminal": true,
    "user": {
      "uid": 0,
      "gid": 0,
      "additionalGids": [
        0,
        1,
        2,
        3,
        4,
        6,
        10,
        11,
        20,
        26,
        27
      ]
    },
    "args": [
      "/bin/sh"
    ],
    "env": [
      "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
      "TERM=xterm",
      "HOME=/root"
    ],
```
