# Use the official Fedora image as the base image
FROM fedora:latest

# Install Skopeo and other necessary packages
RUN dnf -y update && \
    dnf -y install skopeo && \
    dnf install -y git golang make gcc libseccomp-devel libselinux-devel glibc-static && \
    dnf install -y golang     
RUN dnf install -y go-md2man 
RUN git clone https://github.com/opencontainers/umoci.git && cd umoci && make && make install
RUN dnf install -y jq 