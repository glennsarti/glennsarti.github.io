---
title: Getting started with Windows Containers
excerpt: How to get started with Windows Containers, Images and Docker
category:
  - blog
header:
  overlay_image: /images/header-docker.png
  overlay_filter: 0.75
  teaser: /images/teaser-docker.png
tags:
  - windows
  - containers
  - nano
  - docker
---

I wrote a [blog post]({{ site.url }}/blog/neo4j-nano-containers) about running a Neo4j cluster in Windows Containers but I didn't go into too much detail about setting them up.  In this post, I'll walk through setting up Windows Containers and some helpful tips to avoid some problems.

This post is mainly aggregated documentation from Docker, Microsoft, [Stefan Scherer](https://stefanscherer.github.io/) and other sources from when I was installing Windows Containers myself.  Also, this is about Windows Containers, not [Docker for Windows](https://docs.docker.com/docker-for-windows/), which is a different product.
{: .notice--info}

## Before we start - Documentation

Like most things, it's always a good idea to read up on them before diving in...

The documentation both on the [Docker website](https://docs.docker.com/engine/tutorials/usingdocker/) and the [Microsoft website](https://docs.microsoft.com/en-us/virtualization/windowscontainers/) is surprisingly good! It has some good examples and walk throughs as well some in depth documentation.

The [Community and Support](https://docs.microsoft.com/en-us/virtualization/windowscontainers/communitylinks) section also has a list of blog posts and videos for using Windows Containers

As Windows Containers are relatively new, and Linux Containers are not new, a lot of the documenation out there is targetted for Linux Containers.  When searching for help and issues, I found it best to use the `Windows Container` search term instead of `Docker for Windows` to avoid links that were not relevant
{: .notice--warning}

## What are Containers, Images and Docker?

The [Microsoft Windows Containers website](https://docs.microsoft.com/en-us/virtualization/windowscontainers/about/) has a good introduction to Windows Containers

### TLDR version

* Images contain all the filesystem and registry changes for an application to run

* Images can contain other images

* All Images must, eventually, include an OS Image (Server Core or Nano)

* Images are deployed and executed as Containers on Container Hosts

* Many Containers can be deployed from the same Image but are all independent of each other

* Docker wraps up the management of Containers, Images and Container Hosts in an easier to use format


### Container Image
{:.no_toc}

All the file and registry changes required to run an application are packaged into Container Images.  For those that have used Application Virtualisation before (VMware ThinApp, Microsoft AppV etc.), this should sound familiar. An image can include other images so you can layer, or compose, them together, for example, if you have a website that ran on IIS, the website Image would include the IIS image.  Indeed, all images must include at least a Container OS Image.  For Windows Containers this is either the Windows Server Core or Nano Server OS Image.

The name Container Image can be confusing so they're normally just called Images
{: .notice--info}

### Container OS Image
{:.no_toc}

The Container OS image is the first layer in potentially many image layers that make up a container. This image provides the operating system environment. A Container OS Image is Immutable, it cannot be modified.

### Container Host
{:.no_toc}

> Physical or Virtual computer system configured with the Windows Container feature. The container host will run one or more Windows Containers.

### Container
{:.no_toc}

Containers are created when you execute a Container Image, however each Container will have its own copy of the Image which means you can deploy many containers from the same Image safely.

Windows Containers has two isolation models, unlike Linux which has only one.

> Windows Server Containers - provide application isolation through process and namespace isolation technology. A Windows Server container shares a kernel with the container host and all containers running on the host.
>
> Hyper-V Containers – expand on the isolation provided by Windows Server Containers by running each container in a highly optimized virtual machine. In this configuration, the kernel of the container host is not shared with the Hyper-V Containers.

Server 2016 supports both isolation modes, while Windows 10 only supports the Hyper-V Containers. I could not find any documentation on the isolation modes supported by Nano Server but I suspect it is both isolation modes.
{: .notice--warning}

### Docker
{:.no_toc}

Docker is a management and packaging layer on top of Containers.  This means we can use the Docker API on a Linux Container Host or a Windows Container Host with little change.  Also, other tools which build upon Docker, for example, [Docker Compose](https://docs.docker.com/compose/overview/), [Docker Swarm](https://www.docker.com/products/docker-swarm) or [Kubernetes](http://kubernetes.io/docs/getting-started-guides/windows/) can start to be used with Windows Containers.

While the Docker API is the same between Windows and Linux, there are some very different things between them such as volume mounting and networking which I will cover later.
{: .notice--warning}

## What installation options are there?

So how do we install Windows Containers and Docker? This blog post is about creating a development environment so I won't comment on how to deploy Windows Containers in production.

### Windows Containers
{:.no_toc}

For development environments, there are a few options:

- Install locally on Windows 10
- Install locally on Server 2016
- Install a Windows 10 or Server 2016 in Virtual Machine

I personally wouldn't recommend Nano Server for development environment due to how tricky they can be to setup, however it's possible to do.

### Installing locally

The [Microsoft documentation](https://docs.microsoft.com/en-us/virtualization/windowscontainers/quick-start/quick-start-windows-server) is very straightforward and is the same for Server 2016 and Windows 10

#### Windows Server 2016
{:.no_toc}

[Installation instructions](https://docs.microsoft.com/en-us/virtualization/windowscontainers/quick-start/quick-start-windows-server)

``` powershell
PS> Install-Module -Name DockerMsftProvider -Repository PSGallery -Force
PS> Install-Package -Name docker -ProviderName DockerMsftProvider
PS> Restart-Computer -Force
```

#### Windows 10
{:.no_toc}

[Installation instructions](https://docs.microsoft.com/en-us/virtualization/windowscontainers/quick-start/quick-start-windows-10)

``` powershell
PS> Enable-WindowsOptionalFeature -Online -FeatureName containers -All
PS> Enable-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V -All
PS> Restart-Computer -Force

PS> Invoke-WebRequest "https://test.docker.com/builds/Windows/x86_64/docker-1.13.0-rc4.zip" -OutFile "$env:TEMP\docker-1.13.0-rc4.zip" -UseBasicParsing
PS> Expand-Archive -Path "$env:TEMP\docker-1.13.0-rc4.zip" -DestinationPath $env:ProgramFiles
PS> # Add path to this PowerShell session immediately
PS> $env:path += ";$env:ProgramFiles\Docker"
PS> # For persistent use after a reboot
PS> $existingMachinePath = [Environment]::GetEnvironmentVariable("Path",[System.EnvironmentVariableTarget]::Machine)
PS> [Environment]::SetEnvironmentVariable("Path", $existingMachinePath + ";$env:ProgramFiles\Docker", [EnvironmentVariableTarget]::Machine)
PS> dockerd --register-service
PS> Start-Service Docker
```

### Install in a VM

This is basically the same as Installing Locally, but within a Virtual Machine.

### Install in a VM using Vagrant

[Stefan Scherer](https://stefanscherer.github.io/) has created a [Vagrant configuration](https://github.com/StefanScherer/docker-windows-box) which:

> After provisioning the box has the following tools installed:
>
> * Windows Server 2016 with Docker Engine CS 1.12 and client
>
> * docker-machine 0.8.2
>
> * docker-compose 1.9.0
>
> * Docker Tab completion for PowerShell (posh-docker)
>
> * Chocolatey
>
> * Git command line
>
> * Git Tab completion for PowerShell (posh-git)
>
> * SSH client

Which means you can run Windows Containers in both isolation modes _and_ Linux containers (docker-machine)!  Stefan also has some Docker-Swarm examples too which I intend to look into further.


## Docker Client

So now we have a Windows Container Host installed, and a Docker Engine running to manage the Host, now we need the docker client.  Note if you installed Docker on Windows 2016 there should already be a client installed, but you may want to install an updated version.

Fortunately Chocolatey has a docker client package available at [https://chocolatey.org/packages/docker](https://chocolatey.org/packages/docker) which is maintained by [Stefan Scherer](https://stefanscherer.github.io/) and [Ahmet Alp Balkan](https://ahmetalpbalkan.com/)

``` powershell
PS> choco install docker
```

If you would like the most release Docker Client, you can manually download it from the [Github Docker Releases page](https://github.com/docker/docker/releases).  These zip files need to be extracted (like in the Windows 10 installation instructions) before use

### Connecting to remote Docker Hosts

You don't have to install the Docker client on the same host as the Docker server, the Docker API also runs over a network (TCP) connection.

#### Setting up the Docker Server for remote connections

By default, the Docker Server will only listen on a Named Pipe connection, which cannot be accessed remotely.  However, it is fairly easy to add additional listeners:

1. Edit `C:\ProgramData\docker\config\daemon.json`.  If it does not exist, just create the text file

2. Add or modify the hosts setting and append `tcp://0.0.0.0:2375`

``` json
{
    "hosts": ["tcp://0.0.0.0:2375","npipe://"]
}
```

3. Save the file and restart the Docker Service

This configuration will listen on all IP addresses on port 2375 (Default Docker port).  However, if you would like to restrict the manager to, say, a dedicated Management Network Interface (common in production environments), change the IP address appropriately.

More information on configuring the Docker Server on Windows is available on the Microsoft [Windows Containers documentation website](https://docs.microsoft.com/en-us/virtualization/windowscontainers/manage-docker/configure-docker-daemon).

#### Setting up the Docker Client for remote connections

You can specify a remote Docker Server when using the Docker Client in two ways:

1. Use the `-H` or `--host` command line parameter

2. Set the `DOCKER_HOST` environment variable

For example, to connect to a remote Docker Server at IP Address `10.0.0.1` on the default port of `23751

``` powershell
PS> docker -H tcp://10.0.0.1:2375 info

or

PS> $ENV:DOCKER_HOST = 'tcp://10.0.0.1:2375'
PS> docker info
```

To use a local Docker Server over named pipes, use a host of `npipe:////./pipe/docker_engine`.


## Container OS Images

You can start the download of the two Container OS Images by running the appropriate `docker pull` command;

Nano Server Container OS Image (600 MB or so)

``` powershell
PS> docker pull microsoft/nanoserver
```

Windows Server 2016 Server Core Container OS Image (9GB or so)

``` powershell
PS> docker pull microsoft/windowsservercore
```


## File System information

Containers have an isolated file system from the host, which means any file you modify inside the container won't affect the host.  But what about log files or database files how do I mount them inside the container so they can modify them?

When you start a container, you can specify to mount directories on the host inside of the container.  Any modifications to the files will be persisted on the host.

For example;

Let's say I have log directory on my host called `C:\ContainerLogs` and my container writes to log files in the `C:\Logs` directory.  When I start the container, I use a command line similar to below;

``` powershell
PS> docker run -it --volume C:\ContainerLogs:C:\Logs microsoft/nanoserver
...
C:\>dir
 Volume in drive C has no label.
 Volume Serial Number is 7055-B6A6

 Directory of C:\

11/04/2016  09:04 AM             1,894 License.txt
01/14/2017  09:22 PM    <SYMLINKD>     logs [\\?\ContainerMappedDirectories\24E62B19-4E1A-41B8-9B60-26DCC654A55F]
07/16/2016  04:20 AM    <DIR>          Program Files
07/16/2016  04:09 AM    <DIR>          Program Files (x86)
11/04/2016  09:05 AM    <DIR>          Users
01/14/2017  09:23 PM    <DIR>          Windows
               1 File(s)          1,894 bytes
               5 Dir(s)  21,205,929,984 bytes free

C:\>
```

Note that there is a directory called `logs` inside the container and that is a symlink, not a real directory like `Users` or `Windows`.

See the [Docker documentation on volumes]() for more information

### Containers and Images on the Host

By default, the images are stored in `C:\ProgramData\docker\windowsfilter` and containers in `C:\ProgramData\docker\containers`.  To display this information run `docker info` and look for the `Docker Root Dir`

``` powershell
PS> docker info
...
Docker Root Dir: C:\ProgramData\docker
...
```

## Networking Information

Networking within Windows Containers can be a tricky to understand at first.  If you come from a Virtual Machine background, it feels like normal Virtual Switches as used in HyperV (or VMware ESX).  If you come from a Linux Container background, it feels like the usual NAT switches used there.  But in both cases there are some subtle and infuriating differences.

Unlike Linux Containers, Windows Containers offer more networking options and I suggest reading the [Windows Container Networking](https://docs.microsoft.com/en-us/virtualization/windowscontainers/manage-containers/container-networking) document for more detailed infomation.

### Layer 2 Bridge and Layer 2 Tunnel
{:.no_toc}

These are more complex networking options which I don't feel most development environments will take advantage of.

### Network Address Translation (NAT)

This is the most common networking option for Windows Containers.  The Container Host will use the native WinNAT features of the operating system to assign an IP address to Containers and then use port mapping to redirect network traffic appropriately.  However there are some [limitations](https://blogs.technet.microsoft.com/virtualization/2016/05/25/windows-nat-winnat-capabilities-and-limitations/) with the NAT network;

> 1. Only one NAT internal IP prefix is supported per container host, so 'multiple' NAT networks must be defined by partitioning the prefix
>
> 2. Container endpoints are only reachable from the container host using container internal IPs and ports

#### Issue - Only one NAT prefix
{:.no_toc}

WinNAT will only allow one IP prefix (x.x.x.x/y) to be available for translation.  This doesn't mean you can't have multiple NAT based Docker Networks, just that the setup for the WinNAT is more complicated, for example (taken from the documentation);

If you wanted two NAT networks, `172.16.0.0/16` and `172.17.0.0/16`, then the WinNAT prefix must encompass both of those networks: `172.0.0.0/8`, `172.16.0.0/12` or `172.16.0.0/15` are all examples of valid WinNAT prefixes.

##### Creating a custom NAT network
{:.no_toc}

The [Setup NAT Network](https://docs.microsoft.com/en-us/virtualization/hyper-v-on-windows/user-guide/setup-nat-network) documentation describes the setup process in detail.

For this example, I'll walk through the high level steps for setting up two Container NAT Networks (`172.16.0.0/16` and `172.17.0.0/16`) and WinNAT with a prefix of `172.16.0.0/12`, which encompases both of the smaller NAT networks.  I also assume there is already an Internal vSwitch created.

``` powershell
PS> Stop-Service docker
PS> Get-ContainerNetwork | Remove-ContainerNetwork -force
PS> Get-NetNat | Remove-NetNat
```

* Stop the docker service as it will not allow the networks to be modified while running
* Remove the Docker NAT network using the `Remove-ContainerNetwork` cmdlet as this is not possible using the docker client
* Remove the WinNAT configuration, but not remove the associated vSwitch

```
Edit the docker service daemon configuration file `C:\ProgramData\docker\config\daemon.json` and add `bridge: none`.
```

* By default, the Docker service creates a default NAT network, which we don't want.  Add `bridge: none` tells docker not to create the default network


``` powershell
PS> New-ContainerNetwork –Name nat1 –Mode NAT –SubnetPrefix 172.16.0.0/16 -GatewayAddress 172.16.0.1
PS> Get-Netnat | Remove-NetNAT
PS> New-ContainerNetwork –name nat2 –Mode NAT –SubnetPrefix 172.17.0.0/16 -GatewayAddress 172.17.0.1
PS> Get-Netnat | Remove-NetNAT
```
For each NAT network you want to configure

* Create a new NAT container network

* Remove the created WinNAT allocation.  We override that later with our larger prefix

``` powershell
PS> New-NetNat -Name SharedNAT -InternalIPInterfaceAddressPrefix 172.16.0.0/12
PS> Start-Service docker
```
* Configure WinNAT with our larger prefix
* Start the docker service

#### Issue - No localhost
{:.no_toc}

People who have used Linux Containers before will find this issue very quickly.  If you are on the Container Host, you cannot access NAT ports using localhost.  Elton Stoneman has a good writeup in this [blog post](https://blog.sixeyed.com/published-ports-on-windows-containers-dont-do-loopback/).

This is important to remember if you want to do container-to-container networking or you want to test a container resource.

To find the IP Address of container run:

``` powershell
PS> docker inspect --format '{{ "{{" }} .NetworkSettings.Networks.nat.IPAddress }}' <Container ID>
```

Where `<Container ID>` is a tag or ID of running container.

### Transparent Networks

This networking option should be quite familiar.  The container has its own networking stack and appears as a network host in its own right, no hiding behind NAT devices!.  You can create a transparent network using the docker client:

``` powershell
PS> docker network create -d transparent -o com.docker.network.windowsshim.interface="Ethernet 2" TransparentNet2
```

The `-o com.docker.network.windowsshim.interface="Ethernet 2"` binds the transparent network to the network interface called `Ethernet 2` on the Container Host

Assigning IP Addresses to containers can be done via DHCP, however I saw that the Container IP address is not shown when inspecting the container (`docker inspec ....`).  This means, while it's easy to assign IPs the Host cannot coordinate resources when using tools such as Docker Compose.

You can assign Static IPs at container creation time using the `--ip` option, for example

``` powershell
C:\> docker run -it --network=TransparentNet2 --ip 10.123.174.105 microsoft/nanoserver cmdlet
```

This isn't the most scalable solution but it's just enough for development environments.

## Additional Tools

### Portainer

![Portainter Screenshot](http://portainer.io/images/screenshots/portainer-docker-dashboard.jpg){: .align-center}

While the docker client is very powerful, there is always a need for a more graphical view of managing containers.  [Portainer](http://portainer.io/) is a cross platform web based UI for managing a Docker Host, which is very simple to run and use.  Which is great, but they also have a [Windows Container deployment](http://portainer.io/install.html) option and are actively working to make their product work better with Windows Containers.  If you don't want to run Portainer in a container, you can [simply download the application](https://portainer.readthedocs.io/en/stable/deployment.html#without-docker) and run it locally.

It's also open source so you can lodge issues, or even fix bugs yourself!

[Source Code](https://github.com/portainer/portainer)

[Documenation](https://portainer.readthedocs.io/en/stable/)

[Documentation Source](https://github.com/portainer/portainer-docs/)
