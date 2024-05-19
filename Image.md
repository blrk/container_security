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
#### Understanding image layers and sensitive data
* Create a layers directory adn navigate into that directory
```bash
mkdir layers; cd layers
```
* Create index.html
```bash
vi index.html

<html>
<body>
<h1>Hello SCB folks..!! I am inside Container..!!</h1>
</body>
</html>
```
* Create an empty file titled Dockerfile
```bash
vi Dockerfile
```
```bash
FROM ubuntu:22.04
RUN apt update -y
RUN apt install -y apache2
RUN chown -R www-data:www-data /var/www/
RUN echo "some secret" > /password.txt
RUN rm /password.txt
ENV APACHE_RUN_USER www-data
ENV APACHE_RUN_GROUP www-data
ENV APACHE_LOG_DIR /var/log/apache2
ENV APACHE_LOCK_DIR /var/lock/apache2
ENV APACHE_PID_FILE /var/run/apache2.pid
ADD index.html /var/www/html/
EXPOSE 80
ENTRYPOINT ["/usr/sbin/apache2ctl"]
CMD ["-D","FOREGROUND"]
```
* Build a dcoker image
```bash
docker build -t webserver:latest .
```
* List the images
```bash
docker images

REPOSITORY                      TAG       IMAGE ID       CREATED          SIZE
webserver                       latest    776db58f3020   26 seconds ago   237MB
```
* Create a container using the webserver image
```bash
docker run -itd --name webserver -p 80:80 webserver:latest 

c0424b0a884250778c9efec9c3f22a139061c02a32921eb749f13e16f9d53baa
```
* List the running container
```bash
docker ps

CONTAINER ID   IMAGE              COMMAND                  CREATED         STATUS         PORTS                               NAMES
c0424b0a8842   webserver:latest   "/usr/sbin/apache2ct…"   5 seconds ago   Up 4 seconds   0.0.0.0:80->80/tcp, :::80->80/tcp   webserver
```
* Request the webserver to access the default page
```bash
curl http://localhost:80

<html>
<body>
<h1>Hello SCB folks..!! I am inside Container..!!</h1>
</body>
</html>
```
* Login into the container and List the passwod.txt file
```bash
docker exec -it webserver bash

root@c0424b0a8842:/# ls /password/txt
ls: cannot access '/password/txt': No such file or directory

root@c0424b0a8842:/# ls /

bin  boot  dev  etc  home  lib  lib32  lib64  libx32  media  mnt  opt  password.txt  proc  root  run  sbin  srv  sys  tmp  usr  var
```
* exit from the container
* Save the image into the current directory as tar file
```bash
docker save webserver:latest > webserver.tar
```
* List the files
```bash
ls

Dockerfile  index.html  webserver.tar
```
* Create a directory sensitive 
```bash
mkdir sensitive 
```
* extract the tar file into sensitive directory
```bash
tar -xf webserver.tar -C sensitive/
```
* Naviage into sensitive directory and list the contents
```bash
cd sensitive/; ls

blobs  index.json  manifest.json  oci-layout  repositories
```
* List the image layers
```bash
ls blobs/sha256/

06b72af1ca2319aa90802ca7d35125867a02b295d9bbadf5bcac4059b5d04d54  74efce853cfae1d0745950f8d38b6fec927fbabf0a5a49d4811fd6f8414a006c
297a5ca0d9356c02aa7c7620d5d37ab2b7689d7cb73d6c43fedae90db437e4f0  776db58f30204892fa9ef4ccda958e335a054bd42235f404a81bb5c3a4a1f20f
530352e95350190388c540381c1ac6370bbd557e8e579d71336dc385c81f2882  938d9abdbb6c447077d8666b6c9db2115a72062f39c4fc6f82a23dcfc47973ed
629ca62fb7c791374ce57626d6b8b62c76378be091a0daf1a60d32700b49add7  984811d8914a82aa1edf645ed1c0b11c19ddee32d3af87fa94c06482033fe78e
66cc0c5b71750a13bde496c46d43c230c076fd07294bf6e60d67a18191921538  b800a31c01f8f0fe74d6f559c5947f8349d0ff3a6a1fd0a78d303f71f4a1429f
6aa165e259a526586845a92430f8cfb3fa33ad9e1b457629d1b0c3b3647bbd9e  d6b577847a6e31bc2e60ec3ec26828d00e957139c7c01fcb33dc0f977c6d4054
72cff66d5ffca02d6319b38f9c3795fe4f374ac23d7d3ae89eec5b074d344867  e4202b02c9e1c3b554da83810b6ae666374a1b8ce14c5040429c28cf8da4ead0
```
* View the commands used to build the image
```bash
docker history webserver:latest 

IMAGE          CREATED          CREATED BY                                      SIZE      COMMENT
776db58f3020   56 minutes ago   CMD ["-D" "FOREGROUND"]                         0B        buildkit.dockerfile.v0
<missing>      56 minutes ago   ENTRYPOINT ["/usr/sbin/apache2ctl"]             0B        buildkit.dockerfile.v0
<missing>      56 minutes ago   EXPOSE map[80/tcp:{}]                           0B        buildkit.dockerfile.v0
<missing>      56 minutes ago   ADD index.html /var/www/html/ # buildkit        85B       buildkit.dockerfile.v0
<missing>      56 minutes ago   ENV APACHE_PID_FILE=/var/run/apache2.pid        0B        buildkit.dockerfile.v0
<missing>      56 minutes ago   ENV APACHE_LOCK_DIR=/var/lock/apache2           0B        buildkit.dockerfile.v0
<missing>      56 minutes ago   ENV APACHE_LOG_DIR=/var/log/apache2             0B        buildkit.dockerfile.v0
<missing>      56 minutes ago   ENV APACHE_RUN_GROUP=www-data                   0B        buildkit.dockerfile.v0
<missing>      56 minutes ago   ENV APACHE_RUN_USER=www-data                    0B        buildkit.dockerfile.v0
<missing>      56 minutes ago   RUN /bin/sh -c echo "some secret" > /passwor…   12B       buildkit.dockerfile.v0
<missing>      56 minutes ago   RUN /bin/sh -c chown -R www-data:www-data /v…   10.7kB    buildkit.dockerfile.v0
<missing>      56 minutes ago   RUN /bin/sh -c apt install -y apache2 # buil…   108MB     buildkit.dockerfile.v0
<missing>      34 hours ago     RUN /bin/sh -c apt update -y # buildkit         51.6MB    buildkit.dockerfile.v0
<missing>      3 weeks ago      /bin/sh -c #(nop)  CMD ["/bin/bash"]            0B        
<missing>      3 weeks ago      /bin/sh -c #(nop) ADD file:a5d32dc2ab15ff0d7…   77.9MB    
<missing>      3 weeks ago      /bin/sh -c #(nop)  LABEL org.opencontainers.…   0B        
<missing>      3 weeks ago      /bin/sh -c #(nop)  LABEL org.opencontainers.…   0B        
<missing>      3 weeks ago      /bin/sh -c #(nop)  ARG LAUNCHPAD_BUILD_ARCH     0B        
<missing>      3 weeks ago      /bin/sh -c #(nop)  ARG RELEASE                  0B   
```
* find the secret from the specific layer
```bash
cat  blobs/sha256/776db58f30204892fa9ef4ccda958e335a054bd42235f404a81bb5c3a4a1f20f | grep secret | jq '.history'

[
  {
    "created": "2024-04-27T13:18:35.154492474Z",
    "created_by": "/bin/sh -c #(nop)  ARG RELEASE",
    "empty_layer": true
  },
  {
    "created": "2024-04-27T13:18:35.173716357Z",
    "created_by": "/bin/sh -c #(nop)  ARG LAUNCHPAD_BUILD_ARCH",
    "empty_layer": true
  },
  {
    "created": "2024-04-27T13:18:35.206893769Z",
    "created_by": "/bin/sh -c #(nop)  LABEL org.opencontainers.image.ref.name=ubuntu",
    "empty_layer": true
  },
  {
    "created": "2024-04-27T13:18:35.24025223Z",
    "created_by": "/bin/sh -c #(nop)  LABEL org.opencontainers.image.version=22.04",
    "empty_layer": true
  },
  {
    "created": "2024-04-27T13:18:37.180712343Z",
    "created_by": "/bin/sh -c #(nop) ADD file:a5d32dc2ab15ff0d7dbd72af26e361eb1f3e87a0d29ec3a1ceab24ad7b3e6ba9 in / "
  },
  {
    "created": "2024-04-27T13:18:37.512234142Z",
    "created_by": "/bin/sh -c #(nop)  CMD [\"/bin/bash\"]",
    "empty_layer": true
  },
  {
    "created": "2024-05-18T06:18:38.000553674Z",
    "created_by": "RUN /bin/sh -c apt update -y # buildkit",
    "comment": "buildkit.dockerfile.v0"
  },
  {
    "created": "2024-05-19T15:28:11.901505558Z",
    "created_by": "RUN /bin/sh -c apt install -y apache2 # buildkit",
    "comment": "buildkit.dockerfile.v0"
  },
  {
    "created": "2024-05-19T15:28:12.340838341Z",
    "created_by": "RUN /bin/sh -c chown -R www-data:www-data /var/www/ # buildkit",
    "comment": "buildkit.dockerfile.v0"
  },
  {
    "created": "2024-05-19T15:28:12.823372023Z",
    "created_by": "RUN /bin/sh -c echo \"some secret\" > /password.txt # buildkit",
    "comment": "buildkit.dockerfile.v0"
  },
  {
    "created": "2024-05-19T15:28:12.95052592Z",
    "created_by": "ENV APACHE_RUN_USER=www-data",
    "comment": "buildkit.dockerfile.v0",
    "empty_layer": true
  },
  {
    "created": "2024-05-19T15:28:12.95052592Z",
    "created_by": "ENV APACHE_RUN_GROUP=www-data",
    "comment": "buildkit.dockerfile.v0",
    "empty_layer": true
  },
  {
    "created": "2024-05-19T15:28:12.95052592Z",
    "created_by": "ENV APACHE_LOG_DIR=/var/log/apache2",
    "comment": "buildkit.dockerfile.v0",
    "empty_layer": true
  },
  {
    "created": "2024-05-19T15:28:12.95052592Z",
    "created_by": "ENV APACHE_LOCK_DIR=/var/lock/apache2",
    "comment": "buildkit.dockerfile.v0",
    "empty_layer": true
  },
  {
    "created": "2024-05-19T15:28:12.95052592Z",
    "created_by": "ENV APACHE_PID_FILE=/var/run/apache2.pid",
    "comment": "buildkit.dockerfile.v0",
    "empty_layer": true
  },
  {
    "created": "2024-05-19T15:28:12.95052592Z",
    "created_by": "ADD index.html /var/www/html/ # buildkit",
    "comment": "buildkit.dockerfile.v0"
  },
  {
    "created": "2024-05-19T15:28:12.95052592Z",
    "created_by": "EXPOSE map[80/tcp:{}]",
    "comment": "buildkit.dockerfile.v0",
    "empty_layer": true
  },
  {
    "created": "2024-05-19T15:28:12.95052592Z",
    "created_by": "ENTRYPOINT [\"/usr/sbin/apache2ctl\"]",
    "comment": "buildkit.dockerfile.v0",
    "empty_layer": true
  },
  {
    "created": "2024-05-19T15:28:12.95052592Z",
    "created_by": "CMD [\"-D\" \"FOREGROUND\"]",
    "comment": "buildkit.dockerfile.v0",
    "empty_layer": true
  }
]
```
* Note: Anyone who has access to the container image can access any file included in that. From a security prespective storing of sensitive info or token must be avoided in an image. Inside each each imaeg layer's directory there ia another tar file holding the contents of the filesystem at that layer.
 
