# Hands-On Ansible Configuration of Nexus on EC2

This repository contains the necessary files and configurations for setting up a Nexus repository manager on an AWS EC2.

# Pre-requisite
1. Minimum 1 VCPU & 2 GB Memory for the EC2 instance
2. Server firewall opened for port 22 & 8081
3. OpenJDK 8 installed (will be install using ansible)
4. All Nexus processes should run as a non-root nexus user.


---
# Inventory

The inventory file specifies the IP address of the Nexus server which. In my case is EC2 medium.

```inventory 
[nexus]
54.217.x.x
```
NOTE: Ensure you've configured an SSH key for connecting to the server with the designated user to run all tasks as sudors, In my case, I utilized the 'sater' user, which has sudo privileges.

---
# Configuration File
The configuration file ***nexus.yaml*** contains the necessary tasks for setting up the Nexus repository manager as shown here 
[How To Install Latest Sonatype Nexus 3 on Linux](https://devopscube.com/how-to-install-latest-sonatype-nexus-3-on-linux/).
It handles tasks such as installation, configuration, and service management.
- After the final task, the initial admin password will be displayed in the following format:
```
TASK [print initial password for nexus]
ok: [3.251.x.x] => {
    "initial_password.stdout": "xxxx"
}
```
- Copy the password and navigate to `<serverIp>:8081`, then paste it with ***`admin`*** as username
---


## Dockerfile (ubuntu_vm_dockerfile)
- I utilizes a Docker container with Ubuntu as the base image for the Ansible master node instead of my local machine.
- The Dockerfile sets up the necessary dependencies and components required for configuring Ansible to run on it.
- It installs Python, Ansible, and other essential tools.

```Dockerfile
FROM ubuntu:latest

# Update and install necessary dependencies
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
    software-properties-common \
    python3 \
    python3-pip \
    sudo \
    systemd \
    systemd-sysv \
    ansible \
    git 

# Reduce the image size
RUN apt-get clean && \
    rm -rf /var/lib/apt/lists/*

CMD ["/bin/bash"]
```


# Command you will use

1. `docker build -t <imageName> -f ./ .ubuntu_vm_dockerfile .` : to build image 
2. `docker run -it -d <imageName>`                             : create container of that image 
3. `docker exec -it <containerID> bash`                        : enter to the container
4. `git clone <repoURL> `                                      : clone the repo
5. `ansible-playbook -i  <inventoryFileName> --private-key <privateKeyFileName> <configurationFileName.yaml> --ask-become-pass` : to run Ansible playbook.
   > Use `--ask-become-pass` to input the password for the user you've created and connected to on your EC2 instance. This password is required for elevated privileges during the Ansible playbook execution



