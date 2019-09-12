# CyberSecurityDays 2019 - Docker Security Lab

## Overview
In this lab you will get some hand-on experience with Docker. The goal is to take a default Docker Engine installation and secure it as good as recommended for productive usage. Furthermore you will learn how to build a secure Docker image.

If you’re stuck or look for a more detailed definition, refer to the link https://docs.docker.com and go through the official documents of Docker. Please feel free to ask the lab instructor at any time if something is not clear or you are stuck at some point in this lab. 

- ## Agenda
- [Overview](#overview)
- [Agenda](#agenda)
- [Prerequisites](#prerequisites)
- [Lab](#lab)
  - [Part 1 - Docker Engine Setup Verification](#part-1---docker-engine-setup-verification)
  - [Part 2 - Docker Bench for Security](#part-2---docker-bench-for-security)
  - [Part 3 - Docker Runtime Hardening](#part-3---docker-runtime-hardening)
  - [Part 4 - Building Secure Docker Images](#part-4---building-secure-docker-images)
    - [Overview](#overview-1)
    - [Building the Images](#building-the-images)
    - [Start the Voting Application](#start-the-voting-application)

## Prerequisites
If you already downloaded the Hacking-Lab LiveCD you are basically ready and good to go to start this lab. If not, please download the Hacking-Lab LiveCD from https://livecd.hacking-lab.com/ and follow the instructions from https://github.com/ibuetler/e1pub/blob/master/hacking-lab-livecd-installation/install-livecd-en.md to set it up. 
Skip the `VPN` part at the end of the instructions - you do not need the Hacking-Lab OpenVPN for this lab.

**Note:** Its not a MUST to use the LiveCD if you have an own Linux server with Docker installed available. Nevertheless, please be aware that some configuraitons of this lab could break your existing Docker setup if you are using your own server. So please just play with these configuraitons on a **lab** server, if its is not needed for any productive workload.

## Lab
### Part 1 - Docker Engine Setup Verification
In this first lab part you will check the Docker setup inside the Hacking-Lab LiveCD to see if everything is fine and working for this lab.

1. Ensure the Docker Daemon is running and your user is able to run `docker` commands. Open a Terminal window and enter `docker ps` to see the currently running Docker containers (there shouldn’t be any).

2. Now enter the command `docker pull alpine` to pull the `latest` tagged Linux Alpine Docker image. This way we can test, if your Docker daemon inside the Hacking-Lab LiveCD VM is able to reach the Docker Hub registry. If it's not working, please ensure your VirtualBox/VMware Player VM networking settings are correct (for simplicity we recommend using the `NAT` network adapter).

3. Last but not least, check the Docker daemons current settings. At the moment especially the shown ones are interessting for us.
```bash
[root@commander ~]# docker info
...
Runtimes: runc
...
Security Options:
 seccomp
  Profile: default
...
Kernel Version: 3.10.0-957.21.3.el7.x86_64
...
```

### Part 2 - Docker Bench for Security


### Part 3 - Docker Runtime Hardening


### Part 4 - Building Secure Docker Images
#### Overview
Finally its time to dockerize an example application called the "Voting Application". The goal is to build secure Docker images, which can be used for production. The source code of the application itself is provided by Docker (the company).

![architecture](/assets/architecture.png)

#### Building the Images
Start by `git clone` the prepared files to your local machine:
```bash
git clone https://github.com/HSRNetwork/secure-voting-app.git
```

To save some time, we have already prepared the `Dockerfile` for the worker application part. Your job is to resolve all `TODOs` and `…`'s inside the vote and result `Dockerfile`.

Use the official Dockerfile reference in order to get information about the Dockerfile instructions/commands: https://docs.docker.com/engine/reference/builder/

1. 


To build the actual vote/result Docker images use the following commands inside the `./src/vote` and `./src/result` directory (please note the `.` at the end – it's required): 
```bash
docker build -t vote-webapp:1.0 .
docker build -t result-webapp:1.0 .
```
As alternative, use Docker Compose's build feature by simply run `docker-compose build --no-cache`.

#### Start the Voting Application
After successfully building all Docker containers, test if everything starts up by issuing `docker-compose up` inside the same directory where the `docker-compose.yml` file is stored. 
Afterwards, check if http://localhost:5000 shows the vote web application and http://localhost:5001 shows the report web application.

### Part 5 - Optimize your Docker Images (optional)
This last (optional) task is not really security related but it will help you to learn how to build Docker images which are "best practice compliant".
Have a look at the common Dockerfile best practices and improve the Dockerfiles where possible:
https://docs.docker.com/develop/develop-images/dockerfile_best-practices/
