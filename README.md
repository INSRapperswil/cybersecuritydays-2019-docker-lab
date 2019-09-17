# CyberSecurityDays 2019 - Docker Security Lab

## Overview
In this lab you will get some hand-on experience with Docker. The goal is to take a default Docker Engine installation and take some basic steps to secure it. Furthermore you will learn how to build secure Docker images and how to run them in a minimal secure way.

If you’re stuck or look for a more detailed definition, refer to the link https://docs.docker.com and go through the official documents of Docker. Please feel free to ask the lab instructor at any time if something is not clear or you are stuck at some point in this lab.

## Agenda
- [Overview](#overview)
- [Agenda](#agenda)
- [Prerequisites](#prerequisites)
- [Lab](#lab)
  - [Part 1 - Docker Engine Setup Verification](#part-1---docker-engine-setup-verification)
  - [Part 2 - Docker Bench for Security & Docker Networking](#part-2---docker-bench-for-security--docker-networking)
    - [Docker Bench for Security](#docker-bench-for-security)
    - [Docker Networking](#docker-networking)
  - [Part 3 - Network Namespace](#part-3---network-namespace)
  - [Part 4 - Building Secure Docker Images](#part-4---building-secure-docker-images)
    - [Overview](#overview-1)
    - [Building the Images](#building-the-images)
  - [Part 5 - Optimize your Docker Images (Optional)](#part-5---optimize-your-docker-images-optional)

## Prerequisites
If you already downloaded the Hacking-Lab LiveCD you are basically ready and good to go to start this lab. If not, please download the Hacking-Lab LiveCD from https://livecd.hacking-lab.com/ and follow the instructions from https://github.com/ibuetler/e1pub/blob/master/hacking-lab-livecd-installation/install-livecd-en.md to set it up. 
Skip the `VPN` part at the end of the instructions - you do not need the Hacking-Lab OpenVPN for this lab.

**Note:** Its not a MUST to use the LiveCD if you have an own Linux server with Docker installed available. Nevertheless, please be aware that some configuraitons of this lab could break your existing Docker setup if you are using your own server. So please just play with these configurations on a server, if its is not needed for any productive workload.

## Lab
### Part 1 - Docker Engine Setup Verification
In this first lab part you will check the Docker setup inside the Hacking-Lab LiveCD to see if everything is fine and working for this lab.

1. Ensure the Docker daemon is running and your user is able to run `docker` commands. Open a Terminal window and enter `docker ps` to see the currently running Docker containers (there shouldn’t be any).

2. Now enter the command `docker pull alpine` to pull the `latest` tagged Linux Alpine Docker image. This way we can test, if your Docker daemon inside the Hacking-Lab LiveCD VM is able to reach the Docker Hub registry. If it's not working, please ensure your VirtualBox/VMware Player VM networking settings are correct (for simplicity we recommend using the `NAT` network adapter).

3. Last but not least, check the Docker daemons current settings. At the moment, especially the shown ones are interessting for us.
```bash
[root@commander ~]# docker info
...
Runtimes: runc
...
Security Options:
 apparmor
 seccomp
  Profile: default
...
Kernel Version: 5.2.0-kali2-amd64
...
```

### Part 2 - Docker Bench for Security & Docker Networking

#### Docker Bench for Security
Now it's time to run some basic checks against the Docker daemon setup by using the "Docker Bench for Security" Bash script:
```bash
docker run -it --net host --pid host --userns host --cap-add audit_control \
  -e DOCKER_CONTENT_TRUST=$DOCKER_CONTENT_TRUST \
  -v /etc:/etc:ro \
  -v /usr/bin/docker-containerd:/usr/bin/docker-containerd:ro \
  -v /usr/bin/docker-runc:/usr/bin/docker-runc:ro \
  -v /usr/lib/systemd:/usr/lib/systemd:ro \
  -v /var/lib:/var/lib:ro \
  -v /var/run/docker.sock:/var/run/docker.sock:ro \
  docker/docker-bench-security
```

Have a look at the output of this command. As you probably notice there isn't very much to complain about. That's mainly because the Hacking-Lab LiveCD has [Apparmor](https://en.wikipedia.org/wiki/AppArmor) (alternative to SELinux) installed and set to enforcing mode using the `docker-default` profile, Docker Swarm mode is disabled and there aren't many/any Docker images or running container.

#### Docker Networking
Plase have a special look at the following lines from the output of the previous lab part:

```bash
[INFO] 2 - Docker daemon configuration
[WARN] 2.1  - Ensure network traffic is restricted between containers on the default bridge
```

What does this mean? Well it means that any container you start by using `docker run` will be placed inside the default `docker0` Linux bridge and that every container attached to this bridge can commicate with the others via **all** ports! You could place containers inside custom created Docker networks by using the `docker run ` option `--network` but as already said, that's something you have to care about by explicity using this param manually.

Let's prove these statements by running two `network-ninja` containers and test the reachability between each other. The [network ninja](https://github.com/HSRNetwork/network-ninja) Docker image is a simple and relatively small network troubleshooing image with batteries included (some helpful tools installed). We at the INS Intitute for Networked Solutions mainly built and use it to troubleshoot network issues or for lab setups.
Open a second Terminal window and issue the following two statements each inside it's own window.

Terminal Window 1 (server) & 2 (client):
```bash
docker run --rm -it hsrnetwork/network-ninja /bin/bash
```

Now that you should have two open Bash sessions, issue `ip address` inside the server session to get it's current IP address. Write it down, you will need it in a few minutes.

Start a listening service (TCP) on any port you like (e.g. `5000`) by using [Netcat](https://en.wikipedia.org/wiki/Netcat) (`nc`):

```bash
# This is a blocking call. Use Ctrl+c to exit.
nc -l -p 5000
```

On the client side, issue the following Netcad command and send `Hello World` (hit Enter to send it):
```bash
# This is a blocking call. Use Ctrl+c to exit.
nc <server-ip-here> 5000
Hello World<ENTER>
```

Do you receive the message on the server side? You should. If not, please ask an instructor for help.

As you have seen now, communication between these containers inside the `docker0` network can take place via all ports. If you still do not belive it, please try any other port and test the communication again. 
Perhaps with UDP? Netcat parameter `-u` allows you to do so.

Exit the containers by entering `Ctrl+c`.

You could restrict this default communication between containers inside the same Docker network (attached to the same Linux bridge) by setting the Docker daemon property `icc` (inter-container connection) to `false`. This could be done inside `/etc/docker/daemon.json` (file does not exist by default). Afterwards the Docker daemon must be restarted using `systemctl restart docker`.

As an alternative, you could also specify this behaviour on Docker network basis by specifiying `-o "com.docker.network.bridge.enable_icc"="false"`. 
Create the Docker network by using the provided command down here:

```bash
docker network create -o "com.docker.network.bridge.enable_icc"="false" mytest_network
```

Let's start two new containers but this time attached to the just created `mytest_network` Docker network:

Terminal Window 1 (server) & 2 (client):
```bash
docker run --rm -it --net mytest_network hsrnetwork/network-ninja /bin/bash
```

Try again to send a `Hello World` message the same way as before. Does it work? It shouldn't work. Why? Well.. let's have a look at the magic behind Docker's default networking: `iptables`

Get the network ID of the `mytest_network` network:
```bash
root@hlkali:/home/hacker# docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
5a7e554568bd        bridge              bridge              local
c57404655270        host                host                local
891816429ee8        mytest_network      bridge              local
f3be532bd510        none                null                local
```

In this example it's `891816429ee8`. Now look inside the `iptables` `FORWARD` ruleset for the Linux bridge ID of the Docker network (here `891816429ee8`). Search for `br-<your-network-id>` (here `br-891816429ee8`):
```bash
root@hlkali:/home/hacker# iptables -S FORWARD
-P FORWARD DROP
-A FORWARD -j DOCKER-USER
-A FORWARD -j DOCKER-ISOLATION-STAGE-1
-A FORWARD -o br-891816429ee8 -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
-A FORWARD -o br-891816429ee8 -j DOCKER
-A FORWARD -i br-891816429ee8 ! -o br-891816429ee8 -j ACCEPT
-A FORWARD -o docker0 -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
-A FORWARD -o docker0 -j DOCKER
-A FORWARD -i docker0 ! -o docker0 -j ACCEPT
-A FORWARD -i docker0 -o docker0 -j ACCEPT
-A FORWARD -i br-891816429ee8 -o br-891816429ee8 -j DROP
```

As you can see, Docker automatically inserted the `-A FORWARD -i br-891816429ee8 -o br-891816429ee8 -j DROP` rule which drops any traffic from the incoming (`-i`) bridge interface `br-891816429ee8` to itself (`-o`) while for the default bridge interface `docker0` this communication flow is accepted (`-A FORWARD -i docker0 -o docker0 -j ACCEPT`).

Exit the containers by entering `Ctrl+c`.

Lets have another look at Docker's `iptables` handling. Within the server Termin window, start a container which publishes its port `5000` to the outside world via the hosts port `5000`:

```bash
# This is a blocking call. Use Ctrl+c to exit.
docker run --rm -it -p 5000:5000 hsrnetwork/network-ninja nc -l -p 5000
```

Access this port directly from the host via `nc localhost 5000`, send the same message as previouly `Hello World` and once again, hit Enter. You should now see the message inside the server Terminal window.

Finally check whats the magic behind this:

```bash
root@hlkali:/home/hacker# iptables -L DOCKER
Chain DOCKER (1 references)
target     prot opt source               destination         
ACCEPT     tcp  --  anywhere             172.17.0.2           tcp dpt:5000
```

As you can see, Docker simply creates an `iptables` rule inside the `DOCKER` chain which forwards the traffic to `5000/tcp` the the containers IP `172.17.0.2`.

You might now wonder why exactly `iptables`'s `DOCKER` chain? The answer is quite simple, Docker uses multiple `iptables` chains for different purposes:

- `DOCKER`: All Docker rules are loaded to this chain (**e.g. port publishing via `-p`**). It's not recommended to modify this chain manually.
- `DOCKER-USER`: This chain is loaded before `DOCKER`. It allows you to specify custom rules and since the Docker daemon does not mess with this chain, they won't be overridden.
- `DOCKER-ISOLATION-STAGE-1/2`: These two chains isolate the different Docker networks from each other so it's not possible to access `containerX` (running in `networkX`) from `contianerY` (running in `networkY`).

Exit the running container by entering `Ctrl+c`.

### Part 3 - Network Namespace
In this part you will analyze Docker's Linux namespace handling.

1. Start a simple Docker container which does nothing special. Since PID 1 of the container needs to be blocking, just run `ping 8.8.8.8` to keep the container alive (if PID 1 exits, the container is stopped):

```bash
docker run --rm -d alpine ping 8.8.8.8
```

2. Find the just started container's **ID** (save it for later):
```bash
root@hlkali:/home/hacker# docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
0ac5a5a8ab17        alpine              "ping 8.8.8.8"      37 seconds ago      Up 36 seconds                           recursing_feistel
```

3. Get the containers **PID** (replace `0ac5a5a8ab17` with your actual container ID):
```bash
root@hlkali:/home/hacker# docker inspect --format '{{.State.Pid}}' 0ac5a5a8ab17
13758     # <-- That's the containers PID
```

4. Now it's time to enter the containers NET namespace (netns). Use the `nsenter` command down here to start a Bash shell inside the containers netns.
```bash
root@hlkali:/home/hacker# nsenter -t 13758 -n /bin/bash
```
**Hint**: You will not see any output from the `nsenter` command. It just looks like a new Bash line but actually you switched from the default netns to the containers netns.

5. Issue the `ip address` command to see that your Bash shell really entered the containers netns:
```bash
root@hlkali:/home/hacker# ip address
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
49: eth0@if50: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether 02:42:ac:11:00:02 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 172.17.0.2/16 brd 172.17.255.255 scope global eth0
       valid_lft forever preferred_lft forever
```

To compare this IP configuration with the containers IP, start another Terminal window and get it's IP address:
```bash
root@hlkali:/home/hacker# docker exec -it 0ac5a5a8ab17 ip address
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
49: eth0@if50: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue state UP 
    link/ether 02:42:ac:11:00:02 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.2/16 brd 172.17.255.255 scope global eth0
       valid_lft forever preferred_lft forever
```

The IP configuration is identical, right?

Now that you have seen how the Docker daemon set up namespaces in order to isolate the containers from the hosts system and each other, what do you think of the following statement?
```bash
docker run --rm -it --net=host --cap-add=NET_ADMIN alpine /bin/sh
```

You probably guessed right, this container would have **full access to the hosts network**! You better ensure this Docker image is 100% trusted and only really trusted users are member of the `docker` group...


### Part 4 - Building Secure Docker Images
#### Overview
Finally its time to dockerize an example application called the "Voting Application". The goal is to build secure Docker images, which can be used for production. The source code of the application itself is provided by Docker (the company).

![architecture](/assets/architecture.png)

#### Building the Images
Start by `git clone` the prepared files to your local machine:
```bash
git clone https://github.com/HSRNetwork/secure-voting-app.git
```

**Hint:** If you would like to sneak peek into a possible solution for this lab part just take a look at the provided solutions inside the `./secure-voting-app/solution` directory.

To save some time, we have already prepared the `Dockerfile` for the worker application part. Your job is to resolve all `TODOs` and `…`'s inside the vote and result `Dockerfile`.

Use the [official Dockerfile reference](https://docs.docker.com/engine/reference/builder/) in order to get information about the Dockerfile instructions/commands.

**Important:** First of all make sure the latest `docker-compose` version is installed on your Hacking-Lab LiveCD:
```bash
root@hlkali:/home/hacker# apt remove docker-compose -y
root@hlkali:/home/hacker# curl -L "https://github.com/docker/compose/releases/download/1.24.1/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
<...output truncated...>
root@hlkali:/home/hacker# bash
root@hlkali:/home/hacker# docker-compose --version
docker-compose version 1.24.1, build 4667896b
```

Start with the vote `Dockerfile` (`./secure-voting-app/src/vote/Dockerfile`). Eliminate all `TODOs` and `…`'s with the help of the corresponding comments. Try to build the Docker image using the command shown down here to check if your `Dockerfile` is valid:
```bash
cd ./secure-voting-app/src/vote
docker build -t vote-webapp:1.0 .     # Please note the `.` at the end – it's required
```

Now continue with the `Dockerfile` for the result service. Eliminate all `TODOs` and `…`'s with the help of the corresponding comments. Try to build the Docker image using the command shown down here to check if your `Dockerfile` is valid:
```bash
cd ./secure-voting-app/src/result
docker build -t result-webapp:1.0 .     # Please note the `.` at the end – it's required
```

For this service, you should also learn that not every official Docker image is nice and shiny. Have a look at the vulnerability scan from the latest (as of 17.09.2019) `node:current-alpine` image: https://hub.docker.com/layers/node/library/node/current-alpine/images/sha256-7a1789ae7b16137af96748012c6175c0561709f830de29922b7355509f4f9175

![node_current_alpine](/assets/node_current_alpine.png)

Navigate to https://hub.docker.com/_/node?tab=tags and have a look at other images digests (chose architecture `amd64`). Non Linux Alpine based images are most of the time even worse. For example, have a look at the latest (as of 17.09.2019) digest here: https://hub.docker.com/layers/node/library/node/latest/images/sha256-c953b001ea2acf18a6ef99a90fc50630e70a7c0a6b49d774a7aee1f9c937b645

![node_current_alpine_latest](/assets/node_current_alpine_latest.png)

Would you use such a base image for productive usage?

Last but not least, resolve all `TODOs` and `…`'s inside the `./secure-voting-app/docker-compose.yml` file.

**Hint:** To build all services at once which are present inside the `docker-compose.yml` file, use Docker Compose's build feature by simply running `docker-compose build --no-cache`.

To finally test the whole applicaton stack, issue `docker-compose up` and visit http://localhost:5000 and http://localhost:5001 with Firefox or any browser of your choice. 

Open a second Terminal window and check if all processes inside the vote container and all worker processes withing the result container are running with the defined service user. It will probably show UID `hacker` but that's just because the Hacking-Lab LiveCD user `hacker` has the same UID as the service users used inside the containers (UID `1000`).
```bash
root@hlkali:/home/hacker/secure-voting-app/solution# docker top solution_vote_1
UID                 PID                 PPID                C                   STIME               TTY                 TIME                CMD
root                3794                3763                3                   18:09               ?                   00:00:01            /usr/local/bin/python /usr/local/bin/gunicorn app:app -b 0.0.0.0:8080 --user gunicorn-svc-user --group gunicorn-svc-group --log-file - --access-logfile - --workers 4 --keep-alive 0
hacker              4155                3794                1                   18:09               ?                   00:00:00            /usr/local/bin/python /usr/local/bin/gunicorn app:app -b 0.0.0.0:8080 --user gunicorn-svc-user --group gunicorn-svc-group --log-file - --access-logfile - --workers 4 --keep-alive 0
hacker              4159                3794                1                   18:09               ?                   00:00:00            /usr/local/bin/python /usr/local/bin/gunicorn app:app -b 0.0.0.0:8080 --user gunicorn-svc-user --group gunicorn-svc-group --log-file - --access-logfile - --workers 4 --keep-alive 0
hacker              4171                3794                1                   18:09               ?                   00:00:00            /usr/local/bin/python /usr/local/bin/gunicorn app:app -b 0.0.0.0:8080 --user gunicorn-svc-user --group gunicorn-svc-group --log-file - --access-logfile - --workers 4 --keep-alive 0
hacker              4177                3794                1                   18:09               ?                   00:00:00            /usr/local/bin/python /usr/local/bin/gunicorn app:app -b 0.0.0.0:8080 --user gunicorn-svc-user --group gunicorn-svc-group --log-file - --access-logfile - --workers 4 --keep-alive 0
root@hlkali:/home/hacker/secure-voting-app/solution# docker top solution_result_1
UID                 PID                 PPID                C                   STIME               TTY                 TIME                CMD
hacker              4076                4026                3                   18:09               ?                   00:00:01            node server.js
```

To terminate the blocking `docker-compose up` command, simply issue `Ctrl-c`.


### Part 5 - Optimize your Docker Images (Optional)
This last (optional) task is not really security related but it will help you to learn how to build Docker images which are "best practice compliant".
Have a look at the common Dockerfile best practices and improve the Dockerfiles where possible:
https://docs.docker.com/develop/develop-images/dockerfile_best-practices/
