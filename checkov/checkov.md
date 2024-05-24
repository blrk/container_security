### Apply best practices and fix the issues in Dockerfile configuration
#### Scanning and Fixing the issues in Dockerfiles
Objective: To use Checkov to scan and fix Dockerfiles, you can follow these steps. Checkov is a static code analysis tool for infrastructure-as-code (IaC), which includes Dockerfiles, Terraform, CloudFormation, and more.
* Create a container that has Checkov installed
```bash
docker run -itd --name scan blrk/checkov:3.2.106
```
* Get into the container
```bash
docker exec -it scan bash
```
* In the container prompt, Create a Dockerfile using the following code block
```bash
cat > Dockerfile << EOF
FROM ubuntu:latest
RUN apt-get update && apt-get install -y nginx
COPY ./index.html /usr/share/nginx/html/index.html
EXPOSE 8080
EOF
```
* Run the following command to scan the dockerfile
```bash
checkov -f  Dockerfile

FAILED for resource: Dockerfile.FROM
	File: Dockerfile:1-1
	Guide: https://docs.prismacloud.io/en/enterprise-edition/policy-reference/docker-policies/docker-policy-index/ensure-the-base-image-uses-a-non-latest-version-tag

		1 | FROM ubuntu:latest

Check: CKV_DOCKER_2: "Ensure that HEALTHCHECK instructions have been added to container images"
	FAILED for resource: Dockerfile.
	File: Dockerfile:1-4
	Guide: https://docs.prismacloud.io/en/enterprise-edition/policy-reference/docker-policies/docker-policy-index/ensure-that-healthcheck-instructions-have-been-added-to-container-images

		1 | FROM ubuntu:latest
		2 | RUN apt-get update && apt-get install -y nginx
		3 | COPY ./index.html /usr/share/nginx/html/index.html
		4 | EXPOSE 8080

Check: CKV_DOCKER_3: "Ensure that a user for the container has been created"
	FAILED for resource: Dockerfile.
	File: Dockerfile:1-4
	Guide: https://docs.prismacloud.io/en/enterprise-edition/policy-reference/docker-policies/docker-policy-index/ensure-that-a-user-for-the-container-has-been-created

		1 | FROM ubuntu:latest
		2 | RUN apt-get update && apt-get install -y nginx
		3 | COPY ./index.html /usr/share/nginx/html/index.html
		4 | EXPOSE 8080
```
* Try to fix the "ensure-the-base-image-uses-a-non-latest-version-tag" first
* Run the following commnad once again and repeat the scan 
```bash
cat > Dockerfile << EOF
FROM ubuntu:22.04
RUN apt-get update && apt-get install -y nginx
COPY ./index.html /usr/share/nginx/html/index.html
EXPOSE 8080
EOF
```
* Note: Latest version tag is fixed
* Now, fix the "ensure-that-healthcheck-instructions-have-been-added-to-container-images" 
* Modify the dockerfile configuration abd scan it once again
```bash
cat > Dockerfile << EOF
FROM ubuntu:22.04
RUN apt-get update && apt-get install -y nginx
COPY ./index.html /usr/share/nginx/html/index.html
HEALTHCHECK CMD curl --fail http://localhost:8080 || exit 1",
EXPOSE 8080
EOF
```
* Note: The healthcheck issue has been fixed
* Finally, fix the "ensure-that-a-user-for-the-container-has-been-created" 
```bash
cat > Dockerfile << EOF
FROM ubuntu:22.04
RUN apt-get update && apt-get install -y nginx
COPY ./index.html /usr/share/nginx/html/index.html
RUN useradd -ms /bin/bash myuser
USER myuser
HEALTHCHECK CMD curl --fail http://localhost:8080 || exit 1",
EXPOSE 8080
EOF
```
* Note: All the Dockerfile isses has been fixed now. 
* Exit from the container
* Create an image from the container 
* In the host create a 'demo-checkov' and navigate into that
```bash
mkdir ~/demo-checkov; cd demo-checkov
```
* Create a Dockerfile with the previous configuration
```bash
cat > Dockerfile << EOF
FROM ubuntu:22.04
RUN apt-get update && apt-get install -y nginx
COPY ./index.html /usr/share/nginx/html/index.html
RUN useradd -ms /bin/bash myuser
USER myuser
HEALTHCHECK CMD curl --fail http://localhost:8080 || exit 1",
EXPOSE 8080
EOF
```
* Create a index.html page
```bash
cat > ./index.html << EOF
<!DOCTYPE html>
<html>
<body>

<h1>Welcome to Contianer Security course!!!</h1>

</body>
</html>
EOF
```
* Build a docker iamge
```bash
docker build -t mywebserver .
```
* Tag the image
```bash
docker tag  mywebserver blrk/mywebserver:1.0 
```
* Login to docker hub, enter your docker hub credentials 
```bash 
docker login
```
* Push the Docker image
```bash
 docker push blrk/mywebserver:1.0
```

#### Container Image scanning 
* Switch to root user
```bash
sudo su -
```
* Install Trivy 
```bash
curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -s -- -b /usr/local/bin v0.49.1

aquasecurity/trivy info checking GitHub for tag 'v0.49.1'
aquasecurity/trivy info found version: 0.49.1 for v0.49.1/Linux/64bit
aquasecurity/trivy info installed /usr/local/bin/trivy
```
* Exit from the root user
```bash
exit
```
* Scan the mywebserver image push to the dockerhub
```bash
trivy image  blrk/mywebserver:1.0 
2024-05-24T15:23:09.991Z	INFO	Need to update DB
2024-05-24T15:23:09.991Z	INFO	DB Repository: ghcr.io/aquasecurity/trivy-db
2024-05-24T15:23:09.991Z	INFO	Downloading DB...
47.33 MiB / 47.33 MiB [-------------------------------------------------------------------------------------------------------------] 100.00% 1.74 MiB p/s 27s
2024-05-24T15:23:38.869Z	INFO	Vulnerability scanning is enabled
2024-05-24T15:23:38.869Z	INFO	Secret scanning is enabled
2024-05-24T15:23:38.869Z	INFO	If your scanning is slow, please try '--scanners vuln' to disable secret scanning
2024-05-24T15:23:38.869Z	INFO	Please see also https://aquasecurity.github.io/trivy/v0.49/docs/scanner/secret/#recommendation for faster secret detection
2024-05-24T15:23:42.721Z	INFO	Detected OS: ubuntu
2024-05-24T15:23:42.721Z	INFO	Detecting Ubuntu vulnerabilities...
2024-05-24T15:23:42.725Z	INFO	Number of language-specific files: 0

blrk/mywebserver:1.0 (ubuntu 22.04)

Total: 41 (UNKNOWN: 0, LOW: 35, MEDIUM: 6, HIGH: 0, CRITICAL: 0)

┌──────────────────┬────────────────┬──────────┬──────────┬──────────────────────────┬───────────────┬──────────────────────────────────────────────────────────────┐
│     Library      │ Vulnerability  │ Severity │  Status  │    Installed Version     │ Fixed Version │                            Title                             │
├──────────────────┼────────────────┼──────────┼──────────┼──────────────────────────┼───────────────┼──────────────────────────────────────────────────────────────┤
│ coreutils        │ CVE-2016-2781  │ LOW      │ affected │ 8.32-4.1ubuntu1.2        │               │ coreutils: Non-privileged session can escape to the parent   │
│                  │                │          │          │                          │               │ session in chroot                                            │
│                  │                │          │          │                          │               │ https://avd.aquasec.com/nvd/cve-2016-2781                    │
├──────────────────┼────────────────┤          │          ├──────────────────────────┼───────────────┼──────────────────────────────────────────────────────────────┤
│ libgcrypt20      │ CVE-2024-2236  │ MEDIUM   │          │ 1.9.4-3ubuntu3           │               │ libgcrypt: vulnerable to Marvin Attack                       │
│                  │                │          │          │                          │               │ https://avd.aquasec.com/nvd/cve-2024-2236                    │
├──────────────────┼────────────────┼──────────┤          
```
