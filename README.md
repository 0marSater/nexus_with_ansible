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
The configuration file (nexus.yaml) contains the necessary tasks for setting up the Nexus repository manager as shown here 
[How To Install Latest Sonatype Nexus 3 on Linux](https://devopscube.com/how-to-install-latest-sonatype-nexus-3-on-linux/).
It handles tasks such as installation, configuration, and service management.
``` nexus.yaml
- hosts: nexus
  remote_user: sater
  become: yes
  tasks:
    - name: update apt repo
      apt:
        update_cache: yes

    - name: install wget
      apt:
        name: wget
        state: present

    - name: install openjdk 1.8
      apt:
        name: openjdk-8-jdk
        state: present

    - name: create user nexus
      user:
        name: nexus
        shell: /bin/bash
        home: /home/nexus
        createhome: yes

    - name: create directory /app
      file:
        path: /home/nexus/app
        state: directory


    - name: downlaod nexus tar file
      get_url:
        url: https://download.sonatype.com/nexus/3/latest-unix.tar.gz
        dest: /home/nexus/app/nexus.tar.gz
          #force: yes

    - name: untar nexus.tar.gz file
      unarchive:
        remote_src: yes
        src: /home/nexus/app/nexus.tar.gz
        dest: /home/nexus/app/

    - name: Rename the untared file to nexus
      become: true
      command: mv /home/nexus/app/nexus-3.61.0-02 /home/nexus/app/nexus

    - name: change owner of /app/nexus
      file:
        path: /home/nexus/app
        state: directory
        recurse: yes
        owner: nexus
        group: nexus

    - name: uncomment line
      lineinfile:
        path: /home/nexus/app/nexus/bin/nexus.rc
        search_string: '#run_as_user=""'
        line: run_as_user="nexus"

    - name: change nexus data path
      lineinfile:
        path: /home/nexus/app/nexus/bin/nexus.vmoptions
        search_string: '-Dkaraf.data=../sonatype-work/nexus3'
        line: -Dkaraf.data=../nexus/nexus-data

    - name: create nexus service file
      file:
        path: /etc/systemd/system/nexus.service
        state: touch
        owner: nexus
        group: nexus

    - name: Add configuration to nexus service file
      blockinfile:
        path: /etc/systemd/system/nexus.service
        block: |
          [Unit]
          Description=nexus service
          After=network.target

          [Service]
          Type=forking
          LimitNOFILE=65536
          User=nexus
          Group=nexus
          ExecStart=/home/nexus/app/nexus/bin/nexus start
          ExecStop=/home/nexus//app/nexus/bin/nexus stop
          Restart=on-abort

          [Install]
          WantedBy=multi-user.target


    - name: start and enable nexus.service
      service:
        name: nexus
        state: started
        enabled: yes


    - name: wait for the website to become available
      wait_for:
        host: 3.251.89.169
        port: 8081
        state: started
        delay: 5
        timeout: 60

    - name: navigate to nexus website to initiliza admin.password file.
      uri:
        url: http://3.251.89.169:8081/
      register: result
      retries: 5
      delay: 5

    - name: cat initial password for nexus
      command: cat /home/nexus/app/nexus/nexus-data/admin.password
      register: initial_password
      until: result.status==200

    - name: print initial password for nexus
      debug:
        var: initial_password.stdout
```
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



