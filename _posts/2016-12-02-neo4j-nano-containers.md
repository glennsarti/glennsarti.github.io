---
title: Running Neo4j in Windows Containers
excerpt: Using docker-compose to create a three node Neo4j Enterprise cluster on Nano server
category:
  - blog
header:
  image: /images/header-neo4j-globe.png
  overlay_image: /images/header-neo4j-globe.png
  overlay_filter: 0.5
  teaser: /images/teaser-neo-win-docker.png
tags:
  - neo4j
  - powershell
  - docker
  - containers
  - nano
---

![Docker, so hot right now]({{ site.url }}/blog-images/hot-docker-containers.png){: .align-center}

# Docker Docker Docker!

With the release of Windows Server 2016 came Windows Containers.  This is not [Docker-For-Windows](https://docs.docker.com/docker-for-windows/) which is still using a Linux based operating system to run containers, but running containers on Windows with a Windows kernel.

As a bit of an excuse to dive into this new technology from Microsoft, I decided to try running the Neo4j Graph Database within a Windows Container.  Neo4j already release [official docker based images](https://neo4j.com/developer/docker/) but as yet, there are no Windows based containers available.

Disclaimer - This is my first real attempt at using containers so what I created should *only* be considered Proof-of-Concept quality
{: .notice--danger}

### So what are Windows Containers?

Windows Containers are just like their Linux based Containers, but run on Windows kernel.

> What are Containers?
>
> They are an isolated, resource controlled, and portable operating environment.
>
> Basically, a container is an isolated place where an application can run without affecting the rest of the system and without the system affecting the application. Containers are the next evolution in virtualization.
>
> If you were inside a container, it would look very much like you were inside a freshly installed physical computer or a virtual machine. And, to Docker, a Windows Server Container can be managed in the same way as any other container.
>
> [Source - Windows Containers](https://msdn.microsoft.com/en-us/virtualization/windowscontainers/about/index)


# High level steps

* Install Windows Containers

* Create a container for Neo4j (Enterprise v3.0.6)

* Create a three node cluster of Neo4j containers

All source code for this blog is available at;

[https://github.com/glennsarti/code-glennsarti.github.io/tree/master/neo4j-windows-containers](https://github.com/glennsarti/code-glennsarti.github.io/tree/master/neo4j-windows-containers)


## Install Windows Containers

I tried installing Windows Containers on my local Windows 10 laptop, but given I already use a Beta version of the Docker for Windows application, I decided to go the less risky route of quickly creating a Server 2016 VM on my laptop using Hyper-V.  Once I had a VM I merely followed the installation instructions by Microsoft at [https://msdn.microsoft.com/en-us/virtualization/windowscontainers/deployment/deployment](https://msdn.microsoft.com/en-us/virtualization/windowscontainers/deployment/deployment).

My Hyper-V internal network uses 192.168.200.0/24 for it's address.  You will see this throughout the examples below.  The Server 2016 VM was given an IP Address of `192.168.200.50`
{: .notice}

I could then use the docker client on my laptop to access the docker host in the VM simply by setting the environment variable before doing any docker calls.

``` powershell
PS> $ENV:DOCKER_HOST = 'tcp://192.168.200.50:2375'
PS> docker info
Containers: 0
 Running: 0
 Paused: 0
 Stopped: 0
Images: 2
Server Version: 1.12.2-cs2-ws-beta
Storage Driver: windowsfilter
 Windows:
Logging Driver: json-file
Plugins:
 Volume: local
 Network: nat null overlay transparent
Swarm: inactive
Security Options:
Kernel Version: 10.0 14393 (14393.447.amd64fre.rs1_release_inmarket.161102-0100)
Operating System: Windows Server 2016 Standard
OSType: windows
Architecture: x86_64
CPUs: 2
Total Memory: 767 MiB
Name: WIN-07K1J1J7NLN
ID: ********
Docker Root Dir: C:\ProgramData\docker
Debug Mode (client): false
Debug Mode (server): false
Registry: https://index.docker.io/v1/
Insecure Registries:
 127.0.0.0/8
```

There are 2 images because I had already pulled down the `microsoft/nanoserver` and `microsoft/windowsservercore` images
{: .notice}

The docker CLI is the same whether it is accessing a Windows or Linux based container host.


## Creating a DockerFile for Neo4j in a Windows Container

I used the DockerFile from [Neo4j](https://github.com/neo4j/docker-neo4j-publish/blob/master/3.0.6/enterprise/Dockerfile) as the basis for building a image for Windows, but instead used the `microsoft/nanoserver` as the base image. However I came across an issue.  When running `docker build` it was choosing a network which did not have access to the internet thus I couldn't download the required files from within the container.

Fortunately this was an easy problem to solve.  Using the powershell code in the Neo4j Chocolatey Packages as a base, I created a quick PowerShell script which downloaded and extracted a Neo4j Enterprise source Zip file and a Java JRE Tarball.  I also added an entrypoint PowerShell script (`docker-entrypoint.ps1`) which is run when an image started.  These were all placed into a `context` directory and could then be consumed by `docker` during a build.

### `build-neo4jent-context.ps1`

[Source](https://github.com/glennsarti/code-glennsarti.github.io/blob/master/neo4j-windows-containers/build-neo4jent-context.ps1)

This script needs to be run before the docker image can be built.  As stated earlier, it is responsible for downlading the Neo4j Enterprise and Java distributions.  It also sets up the Dockerfile and entrypoint scripts.


### `Dockerfile`

[Source](https://github.com/glennsarti/code-glennsarti.github.io/blob/master/neo4j-windows-containers/Dockerfile)

The base image is `microsoft/nanoserver` because, while there are some Windows Containers which already contain Java available on the Docker Hub, I wanted to build an image from scratch to better understand how they work.
{: .notice}

``` yaml
FROM microsoft/nanoserver

MAINTAINER glennsarti

LABEL Description="Neo4j Enterprise" Vendor="Neo Technologies" Version="3.0.6"

ENTRYPOINT ["powershell.exe","C:/neo4j/docker-entrypoint.ps1"]
CMD ["neo4j"]

COPY neo4j C:/neo4j

# Note - As we're using a transparent network, exposing ports isn't really required.  They are here
# for when we will use a different network driver.

# 7474 = HTTP Neo4j Connector
# 7473 = HTTPS Neo4j Connector
# 7687 = Bolt Connector
# 1337 = Neo4j Shell Port

EXPOSE 7474 7473 7687 1337
```

This is a fairly simple Dockerfile as most of the logic is contained in the `build-neo4jent-context.ps1` and `docker-entrypoint.ps1` scripts

As this container is based on Microsoft Nano server, it is running PowerShell Core inside the container.


### `docker-entrypoint.ps1`

[Source](https://github.com/glennsarti/code-glennsarti.github.io/blob/master/neo4j-windows-containers/docker-entrypoint.ps1)

This entry point file somewhat resembles the [docker-entrypoint.sh](https://github.com/neo4j/docker-neo4j-publish/blob/master/3.0.6/enterprise/docker-entrypoint.sh) bash script used in the offical Neo4j Docker container.

- You pass in the `neo4j` command to run Neo4j

- You pass in the `dump-config` command to display the current neo4j configuration command

- You pass in any other string to run an arbitrary command in the image e.g. `cmd.exe` or `powershell.exe`

- You modify the behaviour of the image by using environment variables:

  *Neo4j HA variables*

  * `NEO4J_dbms_mode`
  * `NEO4J_ha_serverId`
  * `NEO4J_ha_host_data`
  * `NEO4J_ha_host_coordination`
  * `NEO4J_ha_initialHosts`

  *Windows specific*

  * `NEO4J_startup_delay` - Delays the start of the Neo4j Server for the specified number of seconds

For example;

``` powershell
PS> docker run -it -e NEO4J_startup_delay=10 neo4j_enterprise neo4j
```

Would run the container and start a Neo4j single server (not HA) after a delay of 10 seconds


## Container network sadness :-(

[Windows Container Networking](https://msdn.microsoft.com/en-us/virtualization/windowscontainers/management/container_networking?f=255&MSPPError=-2147217396)

### Problem #1 - NAT Networks

Creating a container was simple.  Running a single image and then connecting to the Neo4j Browser was simple.  Creating two images and getting them to form a Neo4j Cluster was a frustrating experience.  But to be fair it was mainly my fault, not Neo4j's or Windows Containters'.  The default networking method in Windows Continers is to use a NAT based gateway.  This means that the internal and external IP address of a Neo4j container are different.  Unfortunately, in Neo4j HA, the configuration parameter used to control the HA server only has one setting: `ha_host_coordination`.  This setting tells Neo4j which network interface to bind to when starting up, and the name of the server that other cluster members will use to contact it.

Neo4j 3.1 addresses this issue and has different settings for `binding` and `advertising` interfaces.
{: .notice--success}

Unfortunately in a NAT setup the inside and outside addresses are different so I had to use a different network type.  Fortunately Windows Containers also have a Transparent network type, which basically creates a virtual network adapter which connects to the hosts external interface.  This can then be given an IP Address via DHCP or staticly assigned via Docker.


### Problem #2 - DHCP

I already have a DHCP server running on my local laptop for my Hyper-V Virtual Machines (Doesn't everyone?) so that bit was done.  I created a new docker network, started a new container on the network and ran `ipconfig`:

``` powershell
PS> docker network create -d transparent TransparentNetwork

PS> docker run -it --network=TransparentNetwork neo4j_enterprise:3.0.6 cmd.exe

C:\> ipconfig

Microsoft Windows [Version 10.0.14393]
(c) 2016 Microsoft Corporation. All rights reserved.

C:\>ipconfig

Windows IP Configuration

Ethernet adapter vEthernet (Container NIC 22c364f7):

   Connection-specific DNS Suffix  . : localdomain
   Link-local IPv6 Address . . . . . : fe80::75c2:3322:7679:a0f4%23
   IPv4 Address. . . . . . . . . . . : 192.168.200.53
   Subnet Mask . . . . . . . . . . . : 255.255.255.0
   Default Gateway . . . . . . . . . : 192.168.200.1
```

Success, my container had an external IP address!

So I then inspected the docker container to make sure it knew what the IP was:

``` powershell
PS> docker inspect <containerid>

...
            "Networks": {
                "TransparentNetwork": {
                    "IPAMConfig": null,
                    "Links": null,
                    "Aliases": [
                        "0d5faefd4669"
                    ],
                    "NetworkID": "18a299ba8b47bfed4245028612bb6207de6803bcce77aae4f2ba24f4c8ed599f",
                    "EndpointID": "eb7fc3a83c728f85cf44dfe72ab86dc38a134e7f212d08dbd3b7755eb3d2aea7",
                    "Gateway": "",
                    "IPAddress": "",
                    "IPPrefixLen": 0,
                    "IPv6Gateway": "",
...
```

Well, it didn't.  Fortunately you can configure the transparent network to use static IP Addresses. So I removed the network I just made and created a new network:

``` powershell
PS> docker network rm TransparentNetwork

PS> docker network create -d transparent --subnet=192.168.200.0/24 --gateway=192.168.200.1 TransparentNetwork
```

And then tried the container on this new network, with a static IP of 192.168.200.100.

``` powershell
PS> docker run -it --network=TransparentNetwork --ip=192.168.200.100 neo4j_enterprise:3.0.6 cmd.exe

C:\> ipconfig

Windows IP Configuration

Ethernet adapter vEthernet (Container NIC a5797484):

   Connection-specific DNS Suffix  . :
   Link-local IPv6 Address . . . . . : fe80::c079:766:6979:ee05%23
   IPv4 Address. . . . . . . . . . . : 192.168.200.100
   Subnet Mask . . . . . . . . . . . : 255.255.255.0
   Default Gateway . . . . . . . . . : 192.168.200.1
```

And then checked the container network in docker

``` powershell
PS> docker inspect <containerid>

...
          "Networks": {
                "TransparentNetwork": {
                    "IPAMConfig": {
                        "IPv4Address": "192.168.200.100"
                    },
                    "Links": null,
                    "Aliases": [
                        "8f1bb6d34fc3"
                    ],
                    "NetworkID": "adc501bbb3b2972f8f87c0b13c38f05d7568b5dc4b2b86bb42e2227f95ab6dac",
                    "EndpointID": "c89a76e6099d1d8224fe02ddbbafeba9c4b4541d66f5d708d78b0d4edaa3e26a",
                    "Gateway": "",
                    "IPAddress": "192.168.200.100",
                    "IPPrefixLen": 24,
                    "IPv6Gateway": "",
...
```

Success!


## Enter docker-compose

So now I had a docker image to run Neo4j, and a network to attach it to, however, I really wanted to create a three node cluster.  While I could just run three docker commands, like Neo4j does in its [documentation](https://neo4j.com/docs/operations-manual/current/deployment/single-instance/docker/#docker-ha) surely there should be a way to define this.  A quick search and I found docker-compose

>Compose is a tool for defining and running multi-container Docker applications. With Compose, you use a Compose file to configure your application’s services. Then, using a single command, you create and start all the services from your configuration.
>
> [Source - Docker Compose](https://docs.docker.com/compose/overview/)

This sounded exactly what I needed! As I had already been playing with Docker for Windows, I already had `docker-compose` installed.  If not you can install it from [Github Release Pages](https://github.com/docker/compose/releases). I quickly created a docker compose configuration file

### `docker-compose.yml`

[Source](https://github.com/glennsarti/code-glennsarti.github.io/blob/master/neo4j-windows-containers/docker-compose.yml)

This simple configuration file will;
- Version 2.1 is compatible with Windows Containers
- Create a three node Neo4j cluser, with one member acting as an arbiter instance
- Use staticly assigned IP addresses
- Will build the neo4j_enterprise contatiner if it doesn't exist already
- Will use the docker network called `TransparentNetwork`
- Adds in dependency information so that the primary docker image must be started before the remaining cluster instances

``` yaml
version: '2.1'

services:

  # Initial cluster member
  neo4j_1:
    build: ./context
    image: neo4j_enterprise:3.0.6
    entrypoint: "powershell C:/neo4j/docker-entrypoint.ps1"
    environment:
      NEO4J_dbms_mode: "HA"
      NEO4J_ha_serverId: "1"
      NEO4J_ha_initialHosts: "192.168.200.201:5001"
    networks:
      neo4jcluster:
        ipv4_address: "192.168.200.201"

  # Additional cluster member
  neo4j_2:
    depends_on:
      - neo4j_1
    image: neo4j_enterprise:3.0.6
    entrypoint: "powershell C:/neo4j/docker-entrypoint.ps1"
    environment:
      NEO4J_startup_delay: "15"
      NEO4J_dbms_mode: "HA"
      NEO4J_ha_serverId: "2"
      NEO4J_ha_initialHosts: "192.168.200.201:5001"
    networks:
      neo4jcluster:
        ipv4_address: "192.168.200.202"

  # Arbiter instance only
  neo4j_3:
    depends_on:
      - neo4j_1
    image: neo4j_enterprise:3.0.6
    entrypoint: "powershell C:/neo4j/docker-entrypoint.ps1"
    environment:
      NEO4J_startup_delay: "15"
      NEO4J_dbms_mode: "ARBITER"
      NEO4J_ha_serverId: "3"
      NEO4J_ha_initialHosts: "192.168.200.201:5001"
    networks:
      neo4jcluster:
        ipv4_address: "192.168.200.203"

# Externally defined transparent network
networks:
  neo4jcluster:
    external:
      name: TransparentNetwork
```

To create the cluster it is simply a matter of running `docker-compose up`.  I've truncated the output so it's easier to read:

``` powershell
PS> docker compose up

Building neo4j_1
Step 1/7 : FROM microsoft/nanoserver
 ---> 787d9f9f8804
...
Step 6/7 : COPY neo4j C:/neo4j
Step 7/7 : EXPOSE 7474 7473 7687 1337
 ---> Running in b59d5c78cb77
 ---> 811e3a6e8e18
Removing intermediate container b59d5c78cb77
Successfully built 811e3a6e8e18
WARNING: Image for service neo4j_1 was built because it did not already exist. To rebuild this image you must use `docker-compose build` or `docker-compose up --build`.
Creating neo4jwindowscontainers_neo4j_1_1
Creating neo4jwindowscontainers_neo4j_2_1
Creating neo4jwindowscontainers_neo4j_3_1
Attaching to neo4jwindowscontainers_neo4j_1_1, neo4jwindowscontainers_neo4j_2_1, neo4jwindowscontainers_neo4j_3_1
Attaching to neo4jwindowscontainers_neo4j_1_1, neo4jwindowscontainers_neo4j_2_1, neo4jwindowscontainers_neo4j_3_1
neo4j_2_1  | Container IP Address is 192.168.200.202
neo4j_3_1  | Container IP Address is 192.168.200.203
neo4j_1_1  | Container IP Address is 192.168.200.201
neo4j_2_1  | Detected environment variable NEO4J_dbms_mode
...
neo4j_3_1  | 2016-12-04 04:32:50.522+0000 INFO  [o.n.c.c.ClusterJoin] Attempting to join cluster of [192.168.200.201:5001]
neo4j_1_1  | 2016-12-04 04:32:50.770+0000 INFO  Could not join cluster of [192.168.200.201:5001]
neo4j_1_1  | 2016-12-04 04:32:50.832+0000 INFO  Creating new cluster with name [neo4j.ha]...
neo4j_1_1  | 2016-12-04 04:32:50.963+0000 INFO  Instance 1 (this server)  joined the cluster
neo4j_1_1  | 2016-12-04 04:32:51.256+0000 INFO  Instance 1 (this server)  was elected as coordinator
neo4j_1_1  | 2016-12-04 04:32:51.488+0000 INFO  Instance 1 (this server)  was elected as coordinator
neo4j_1_1  | 2016-12-04 04:32:51.614+0000 INFO  I am 1, moving to master
neo4j_2_1  | 2016-12-04 04:32:52.020+0000 INFO  Starting...
neo4j_1_1  | 2016-12-04 04:32:52.112+0000 INFO  Instance 3  joined the cluster
neo4j_3_1  | 2016-12-04 04:32:52.133+0000 INFO  [o.n.c.c.ClusterJoin] Joined cluster: Name:neo4j.ha Nodes:{1=cluster://192.168.200.201:5001, 3=cluster://192.168.200.203:5001} Roles:{coordinator=1}
neo4j_1_1  | 2016-12-04 04:32:52.568+0000 INFO  I am 1, successfully moved to master
neo4j_1_1  | 2016-12-04 04:32:52.754+0000 INFO  Instance 1 (this server)  is available
...
neo4j_1_1  | 2016-12-04 04:33:02.597+0000 INFO  Instance 1 (this server)  was elected as coordinator
neo4j_2_1  | 2016-12-04 04:33:02.605+0000 INFO  Instance 1  was elected as coordinator
neo4j_1_1  | 2016-12-04 04:33:02.747+0000 INFO  Instance 1 (this server)  is available as master at ha://192.168.200.201:6001?serverId=1 with StoreId{creationTime=1480825964492, randomId=1207142648701028915, storeVersion=15531981201765894, upgradeTime=1480825964492, upgradeId=1}
neo4j_1_1  | 2016-12-04 04:33:02.853+0000 INFO  Instance 1 (this server)  is available as backup at backup://127.0.0.1:6362 with StoreId{creationTime=1480825964492, randomId=1207142648701028915, storeVersion=15531981201765894, upgradeTime=1480825964492, upgradeId=1}
neo4j_2_1  | 2016-12-04 04:33:02.892+0000 INFO  Instance 1  is available as master at ha://192.168.200.201:6001?serverId=1 with StoreId{creationTime=1480825964492, randomId=1207142648701028915, storeVersion=15531981201765894, upgradeTime=1480825964492, upgradeId=1}
neo4j_2_1  | 2016-12-04 04:33:03.054+0000 INFO  Instance 1  is available as backup at backup://127.0.0.1:6362 with StoreId{creationTime=1480825964492, randomId=1207142648701028915, storeVersion=15531981201765894, upgradeTime=1480825964492, upgradeId=1}
neo4j_2_1  | 2016-12-04 04:33:03.565+0000 INFO  ServerId 2, moving to slave for master ha://192.168.200.201:6001?serverId=1
neo4j_1_1  | 2016-12-04 04:33:03.634+0000 INFO  Started.
neo4j_2_1  | 2016-12-04 04:33:04.070+0000 INFO  Checking store consistency with master
neo4j_2_1  | 2016-12-04 04:33:04.080+0000 INFO  The store does not represent the same database as master. Will remove and fetch a new one from master
neo4j_1_1  | 2016-12-04 04:33:04.218+0000 INFO  Instance 2  is unavailable as backup
neo4j_2_1  | 2016-12-04 04:33:04.219+0000 INFO  Instance 2 (this server)  is unavailable as backup
...
neo4j_2_1  | 2016-12-04 04:33:06.048+0000 INFO  Copying schema\label\lucene\labelStore\1\segments_1
neo4j_2_1  | 2016-12-04 04:33:06.626+0000 INFO  Copied schema\label\lucene\labelStore\1\segments_1 71.00 B
neo4j_2_1  | 2016-12-04 04:33:06.637+0000 INFO  Done, copied 17 files
neo4j_1_1  | 2016-12-04 04:33:07.753+0000 INFO  Remote interface available at http://0.0.0.0:7474/
neo4j_2_1  | 2016-12-04 04:33:10.236+0000 INFO  Finished copying store from master
neo4j_2_1  | 2016-12-04 04:33:10.307+0000 INFO  Checking store consistency with master
...
neo4j_2_1  | 2016-12-04 04:33:16.086+0000 DEBUG Mounting servlet at [/db/manage]
neo4j_2_1  | 2016-12-04 04:33:16.126+0000 DEBUG Mounting servlet at [/db/data]
neo4j_2_1  | 2016-12-04 04:33:16.182+0000 DEBUG Mounting servlet at [/]
neo4j_2_1  | 2016-12-04 04:33:17.887+0000 INFO  Remote interface available at http://0.0.0.0:7474/
```

One of the unexpected things with docker-compose is that all of the output from all three servers is aggregate so you can easily see what happens when a node joins the cluster.

### Simulating a cluster node failure

We can easily emulate a cluster node failure by stopping one of the containers.

``` powershell
PS> docker ps
CONTAINER ID        IMAGE                    COMMAND                  CREATED             STATUS              PORTS                               NAMES
4efbfaf7683b        neo4j_enterprise:3.0.6   "powershell.exe C:..."   6 minutes ago       Up 6 minutes        1337/tcp, 7473-7474/tcp, 7687/tcp   neo4jwindowscontainers_neo4j_3_1
19941e8d0ea1        neo4j_enterprise:3.0.6   "powershell.exe C:..."   6 minutes ago       Up 6 minutes        1337/tcp, 7473-7474/tcp, 7687/tcp   neo4jwindowscontainers_neo4j_2_1
fa179e91dab3        neo4j_enterprise:3.0.6   "powershell.exe C:..."   6 minutes ago       Up 6 minutes        1337/tcp, 7473-7474/tcp, 7687/tcp   neo4jwindowscontainers_neo4j_1_1
C:\Source> docker stop neo4jwindowscontainers_neo4j_1_1
neo4jwindowscontainers_neo4j_1_1
PS>
```

In the `docker-compose` output we see:

``` powershell
neo4jwindowscontainers_neo4j_1_1 exited with code 1067
neo4j_2_1  | 2016-12-04 04:38:16.907+0000 INFO  Instance 1  has failed
neo4j_2_1  | 2016-12-04 04:38:17.009+0000 INFO  Instance 2 (this server)  was elected as coordinator
neo4j_2_1  | 2016-12-04 04:38:17.031+0000 INFO  I am 2, moving to master
neo4j_2_1  | 2016-12-04 04:38:17.347+0000 INFO  I am 2, successfully moved to master
neo4j_2_1  | 2016-12-04 04:38:17.371+0000 INFO  Instance 2 (this server)  is available as master at ha://192.168.200.202:6001?serverId=2 with StoreId{creationTime=1480825964492, randomId=1207142648701028915, storeVersion=15531981201765894, upgradeTime=1480825964492, upgradeId=1}
neo4j_2_1  | 2016-12-04 04:38:18.309+0000 INFO  Instance 2 (this server)  is available as backup at backup://127.0.0.1:6362 with StoreId{creationTime=1480825964492, randomId=1207142648701028915, storeVersion=15531981201765894, upgradeTime=1480825964492, upgradeId=1}
```

The container stopped and then neo4j_2_1 was re-elected to master

We can then restart the failed node

``` powershell
PS> docker restart neo4jwindowscontainers_neo4j_1_1
neo4jwindowscontainers_neo4j_1_1
PS>
```

In the `docker-compose` output we see:

``` powershell
neo4j_1_1  | Container IP Address is 192.168.200.201
neo4j_1_1  | Detected environment variable NEO4J_dbms_mode
neo4j_1_1  | Detected environment variable NEO4J_ha_serverId
neo4j_1_1  | Detected environment variable NEO4J_ha_initialHosts
neo4j_1_1  | VERBOSE: Neo4j Root is 'C:\neo4j'
neo4j_1_1  | VERBOSE: Neo4j Server Type is 'Enterprise'
neo4j_1_1  | VERBOSE: Neo4j Version is '3.0.6'
...
neo4j_1_1  | 2016-12-04 04:32:50.963+0000 INFO  Instance 1 (this server)  joined the cluster
neo4j_1_1  | 2016-12-04 04:32:51.256+0000 INFO  Instance 1 (this server)  was elected as coordinator
neo4j_1_1  | 2016-12-04 04:32:51.488+0000 INFO  Instance 1 (this server)  was elected as coordinator
neo4j_1_1  | 2016-12-04 04:32:51.614+0000 INFO  I am 1, moving to master
neo4j_1_1  | 2016-12-04 04:32:52.112+0000 INFO  Instance 3  joined the cluster
neo4j_1_1  | 2016-12-04 04:32:52.568+0000 INFO  I am 1, successfully moved to master
...
neo4j_1_1  | 2016-12-04 04:33:07.753+0000 INFO  Remote interface available at http://0.0.0.0:7474/
neo4j_1_1  | 2016-12-04 04:33:10.663+0000 INFO  Instance 2  is available as slave at ha://192.168.200.202:6001?serverId=2 with StoreId{creationTime=1480825964492, randomId=1207142648701028915, storeVersion=15531981201765894, upgradeTime=1480825964492, upgradeId=1}
```

The container restarted, and was re-elected to master

### Cleaning up

Once we finished with our containers, simply press `Ctrl-C` and docker-compose will stop all containers, and then `docker-compose down` will cleanup any created containers, volumes or networks.  It will not remove images though.

``` powershell
Gracefully stopping... (press Ctrl+C again to force)
Stopping neo4jwindowscontainers_neo4j_3_1 ... done
Stopping neo4jwindowscontainers_neo4j_2_1 ... done
Stopping neo4jwindowscontainers_neo4j_1_1 ... done
PS> docker-compose down
Removing neo4jwindowscontainers_neo4j_3_1 ... done
Removing neo4jwindowscontainers_neo4j_2_1 ... done
Removing neo4jwindowscontainers_neo4j_1_1 ... done
Network TransparentNetwork is external, skipping
PS>
```

## Conclusion

After a bit of a battle with networking, I can now quickly spin up a three node Neo4j cluster, with an arbiter instance, all on Windows and then destroy it easily when I'm done!

Thanks to [Ben Butler-Cole](https://github.com/benbc) and [Mark Needham](https://twitter.com/markhneedham) for helping me out with some Neo4j issues.
