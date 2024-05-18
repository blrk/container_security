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
#### set up SELinux policies for Docker on an Amazon Linux 3 instance
* Ensure SELinux is enabled and set to enforcing mode
```bash
sudo setenforce 1
sudo sed -i 's/^SELINUX=.*$/SELINUX=enforcing/' /etc/selinux/config
```
* Check the SELinux configuration
```bash
sestatus

SELinux status:                 enabled
SELinuxfs mount:                /sys/fs/selinux
SELinux root directory:         /etc/selinux
Loaded policy name:             targeted
Current mode:                   enforcing
Mode from config file:          enforcing
Policy MLS status:              enabled
Policy deny_unknown status:     allowed
Memory protection checking:     actual (secure)
Max kernel policy version:      33
```
* Install necessary packages
```bash
sudo yum install -y container-selinux policycoreutils-python-utils
```
* Create a container to mount the /var/log directory from the host directory
```bash
docker run -d -v /var/log:/var/www/html --name mywebapp_container1 httpd:latest

59dae04ef6b4f24f5f1b1b25ab2dacbc55757c72fcb1ed8684b2bdd90b06d3c0
```
* Get into the container and navigate into the mount path and list the files
```bash
docker exec -it 59d bash

root@59dae04ef6b4:/usr/local/apache2# cd /var/www/html/
root@59dae04ef6b4:/var/www/html# ls
ls: cannot open directory '.': Permission denied
```
* Note: You get blocked, even though running as the root user. 
* To confirm that SELinux is blocking access, you can run the same container and add the --security-opt label:disable to the command, which effectively disables SELinux for that container. If we then try creating a file inside the /var/log directory, you can see that it is successful.
```bash
docker run -d -v /var/log:/var/www/html --name mywebapp_container2 --security-opt label:disable httpd:latest

35675db03cc39087de24f68d94543869aa6dc15def8756eb1792f0a89bd4c7d2
```
* Get into the container and navigate into the mount path, list the files and try to create a new file
``` bash
docker exec -it 356 bash
```
```bash
root@35675db03cc3:/usr/local/apache2# cd /var/www/html/
root@35675db03cc3:/var/www/html# ls

README	amazon	audit  btmp  chrony  cloud-init-output.log  cloud-init.log  dnf.librepo.log  dnf.log  dnf.rpm.log  hawkey.log  hawkey.log-20240516  journal  lastlog  private  sa  tallylog  wtmp
```bash
root@35675db03cc3:/var/www/html# touch myfile
```
``` bash
ls

README	amazon	audit  btmp  chrony  cloud-init-output.log  cloud-init.log  dnf.librepo.log  dnf.log  dnf.rpm.log  hawkey.log  hawkey.log-20240516  journal  lastlog  myfile  private  sa  tallylog  wtmp
```
* exit form the container
#### Creating Custom SELinux Policy for Docker
* Suppose you have a web application running in a Docker container that needs to access a directory on the host system. Follow these steps to create and apply a custom SELinux policy
* Create a directory for the web application
```bash
sudo mkdir -p /srv/mywebapp
sudo chown -R 1000:1000 /srv/mywebapp  # Assuming container uses UID 1000
```
* Create a directory and navigate into that 
```bash
mkdir selinux; cd selinux/
```
* Create a custom policy module
```bash
vi mywebapp.te
```
* Add the following configuration
```bash
module mywebapp 1.0;

require {
    type container_t;
    type var_t;
    class dir { read write add_name remove_name search open };
    class file { read write create getattr setattr lock append unlink link rename execute open };
}

# Allow container_t to access /srv/mywebapp directory
allow container_t var_t:dir { read write add_name remove_name search open };
allow container_t var_t:file { read write create getattr setattr lock append unlink link rename execute open };
```
* Compile and load the SELinux policy
```bash
checkmodule -M -m -o mywebapp.mod mywebapp.te
semodule_package -o mywebapp.pp -m mywebapp.mod
sudo semodule -i mywebapp.pp
```
* Label the directory with the correct SELinux context
```bash
sudo semanage fcontext -a -t var_t "/srv/mywebapp(/.*)?"
sudo restorecon -R /srv/mywebapp
```
* Run the Docker container with SELinux enforced
```bash
docker run -d -v /srv/mywebapp:/var/www/html --name mywebapp_container httpd:latest

432bba0fd66712cf49d78545aa8ef2339c76aa9d2d76c42a9a2aac6a55b0875f
```
* 
```bash
docker exec -it 432 /bin/bash

root@432bba0fd667:/usr/local/apache2# cd /var/www/html/

root@432bba0fd667:/var/www/html# ls

root@432bba0fd667:/var/www/html# touch index.html

root@432bba0fd667:/var/www/html# ls

index.html
```
* Exit from the container
* List the webapp directory of the host
```bash
[ec2-user@container-sec selinux]$ ls /srv/mywebapp/

index.html
```
* If we want to create a custom SELinux policy that will allow us to perform custom actions from container, one option is to use a tool like udica.
* https://github.com/containers/udica

#### Troubleshooting SELinux Issues
* Check audit logs
```bash
sudo ausearch -m avc -ts recent
```
* Generate a custom policy for denials
```bash
sudo ausearch -m avc -ts recent | audit2allow -M mypol
sudo semodule -i mypol.pp
```
#### extra Managing SELinux Booleans for Docker
* Allow Docker to make the container runtime executable:
```bash
sudo setsebool -P container_manage_cgroup on
```
* Allow Docker to use host's network stack:
```bash
sudo setsebool -P container_use_execmem on
```