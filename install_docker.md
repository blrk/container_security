* Install Docker

sudo yum update -y

sudo yum install -y docker

sudo service docker start

sudo usermod -a -G docker ec2-user

* Logout and login again

docker ps

