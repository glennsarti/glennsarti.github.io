---
title: Puppet Server in Docker on Windows
excerpt: Running Puppet Server in Docker on Windows for Windows
category:
  - blog
header:
  overlay_image: /images/header-docker.png
  overlay_filter: 0.75
  teaser: /images/teaser-puppet-in-docker.png
tags:
  - puppet
  - docker
  - windows
  - compose
---

# Puppet Server, in Docker, on Windows

I've found that time to time, I really need a Puppet Server to do some testing.  For example when I'm testing a Puppet control repository or that a module behaves correctly.  Now I could use Vagrant to spin up a VM, or use a preconfigured Azure VM, but I wanted something that worked better with software development.  All the cool kids are using docker so why not use docker to run Puppet Server?

[Gareth Rushgrove](https://twitter.com/garethr) did some initial work on creating [puppet containers](https://hub.docker.com/u/puppet/) but it wasn't until [Michael Stahnke](https://twitter.com/stahnma) mentioned his [puppetstack](https://github.com/stahnma/puppetstack) repository to me that I realised that this should be somewhat easy to do.

# What's docker and docker-compose?

I blogged previously about [Docker, Containers](../getting-started-with-windows-containers) and [Docker Compose](../neo4j-nano-containers).

# Starting out ...

So Michael's [original docker-compose file](https://github.com/stahnma/puppetstack/blob/0339495513f8378e27b6991b03bdfa56cdc93082/docker-compose.yml) was geared towards Docker on Mac or Linux.  In particular it referenced;

``` yaml
volumes:
     - /var/run/docker.sock:/var/run/docker.sock
```

The docker unix socket is not available on Windows.  So straight away this wouldn't work on Docker for Windows but we could make some changes to get this to work.

## Removing the load balancer

The docker unix socket is being used by the `gobetween-puppetserver` service.  [GoBetween](http://gobetween.io/) is small Layer4 proxy server which is being used to balance requests to Puppet Servers.  As I was just creating a simple Puppet Server configuration, I only need one Puppet Server container which means I didn't need a proxy server.  So in this case I just removed the `gobetween-puppetserver` service entirely, and removed the dependency in the `puppet` service.  This meant I could now create the docker containers and see what else broke!

## Postgres blues ....

So I started docker-compose and immediately saw some errors;

``` powershell

PS> docker-compose up

puppetexplorer_1    | Activating privacy features... done.
puppetexplorer_1    | http://
postgres            | The files belonging to this database system will be owned by user "postgres".
postgres            | This user must also own the server process.
postgres            |
postgres            | The database cluster will be initialized with locale "en_US.utf8".
postgres            | The default database encoding has accordingly been set to "UTF8".
postgres            | The default text search configuration will be set to "english".
...
postgres            | waiting for server to start....FATAL:  data directory "/var/lib/postgresql/data" has wrong ownership
postgres            | HINT:  The server must be started by the user that owns the data directory.
postgres            |  stopped waiting
postgres            | pg_ctl: could not start server
...
```

A quick search and I found other people had experienced this before, and in hindsight it makes perfect sense.  The compose file used a volume mapping for the PostGres data;

``` yaml
    volumes:
      - puppetdb-postgres-volume:/var/lib/postgresql/data/
```

Under Docker For Windows the bind mount volumes are owned by root, but PostgreSQL is expecting the owner to be postgres, and will fail unless this is true.  But a [forum post](https://forums.docker.com/t/trying-to-get-postgres-to-work-on-persistent-windows-mount-two-issues/12456/4) had a nice workaround.  Instead of using a bind mount, use a volume.  The volume is created within Docker's storage, instead of a bind mount which is doing file redirection.  That means the ownership problem is bypassed, but it means the volume has to be created before docker compose, and cleaned up as well.

So I changed the docker-compose file with;

``` yaml
services:

  ...

  puppetdbpostgres:

    ...

    volumes:
      - puppetdb-postgres-volume:/var/lib/postgresql/data/

...

volumes:
  puppetdb-postgres-volume:
    external: true
```

This instructed docker to find a volume called `puppetdb-postgres-volume` and mount that in the `puppetdbpostgres` as `/var/lib/postgresql/data`.

So to create the volume we use;

``` powershell
PS> docker volume create --name puppetdb-postgres-volume -d local
```

And to remove the volume we use;

``` powershell
PS> docker volume rm puppetdb-postgres-volume
```

## Success

So with PostgreSQL fixed it was time to try the puppet stack again;

``` powershell
PS> docker volume create --name puppetdb-postgres-volume -d local
puppetdb-postgres-volume
PS> docker-compose up
...
LOTS and LOTS of output
...
puppetboard_1       | 127.0.0.1 - - [25/Apr/2018:05:26:51 +0000] "GET / HTTP/1.1" 404 3312 "-" "curl/7.59.0"
puppetdb_1          | 172.18.0.2 - - [25/Apr/2018:05:26:51 +0000] "GET /pdb/query/v4/environments HTTP/1.1" 200 2 "-" "-"
puppetdb_1          | 172.18.0.2 - - [25/Apr/2018:05:26:51 +0000] "GET /pdb/query/v4/environments HTTP/1.1" 200 2 "-" "-"
puppetdb_1          | 172.18.0.2 - - [25/Apr/2018:05:26:51 +0000] "GET /pdb/query/v4/environments HTTP/1.1" 200 2 "-" "-"
puppetboard_1       | 127.0.0.1 - - [25/Apr/2018:05:26:51 +0000] "GET / HTTP/1.1" 404 3312 "-" "curl/7.59.0"
puppet_1            | 127.0.0.1 - - - 25/Apr/2018:05:26:56 +0000 "GET /production/status/test HTTP/1.1" 200 35 127.0.0.1 127.0.0.1 8140 13
```

After all of the setup errors and warnings stopped being spewed out, eventually the logs were just showing various health check REST endpoints being hit e.g. `"GET /production/status/test HTTP/1.1"`.  So now it was time to see if the environment was actually working.  I fired up Chrome and browsed to `http://127.0.0.1`, which is the endpoint for [Puppet Explorer](https://github.com/dalen/puppetexplorer), and crossed my fingers ...

![Puppet Explorer Success]({{ site.url }}/blog-images/puppet-docker-001.png){: .align-center}

Success!  What about [Puppet Board](https://github.com/voxpupuli/puppetboard)?  It was being published on port 8000 so I browsed to `http://127.0.0.1:8000` and crossed my fingers ...

![Puppet Board Success]({{ site.url }}/blog-images/puppet-docker-002.png){: .align-center}

Somewhat success.  As there were no nodes attached to the master, of course there was no data in PuppetDB.  So the next thing I needed to do was attach an agent to the master.

# Adding an agent

I fired up a Windows VM, and installed a puppet agent using chocolatey (`choco install puppet-agent`).  I then modified the hosts file, creating a entry called puppet to my laptop's external IP Address.  In my case it was 192.168.100.105;

``` powershell
PS> "`n192.168.100.105 puppet" | Out-File C:\windows\system32\drivers\etc\hosts -Append -Encoding UTF8
```

I then executed puppet in agent mode to start the certificate process;

``` powershell
PS> puppet agent -t --noop
Error: Could not request certificate: Failed to open TCP connection to puppet:8140 (A connection attempt failed because the connected party did not properly respond after a period of time, or established connection failed because connected host has failed to respond. - connect(2) for "puppet" port 8140)
Exiting; failed to retrieve certificate and waitforcert is disabled

PS> ping puppet

Pinging puppet [192.168.100.105] with 32 bytes of data:
Reply from 192.168.100.105: bytes=32 time<1ms TTL=128
Reply from 192.168.100.105: bytes=32 time<1ms TTL=128
Reply from 192.168.100.105: bytes=32 time<1ms TTL=128
Reply from 192.168.100.105: bytes=32 time<1ms TTL=128

Ping statistics for 192.168.100.105:
    Packets: Sent = 4, Received = 4, Lost = 0 (0% loss),
Approximate round trip times in milli-seconds:
    Minimum = 0ms, Maximum = 0ms, Average = 0ms
PS C:\Users\Administrator>
```

Well that wasn't great.  I could ping `puppet` and it resolved to the correct IP Address so perhaps it the TCP port wasn't listening.  I had a look at docker-compose file and noticed something;

``` yaml
services:
  puppet:

    ...

    ports:
      - 8140
```

The file says that port 8140 is exposed but there was no mapping so it ended up on some random port.  Time to inspect the container

``` powershell
PS> docker ps
CONTAINER ID        IMAGE                      COMMAND                  CREATED             STATUS                      PORTS                                              NAMES
e5476faaf9f2        puppet/puppetserver        "dumb-init /docker-e…"   31 minutes ago      Up 31 minutes (healthy)     0.0.0.0:32779->8140/tcp                            singlehost_puppet_1
2596c50079ba        puppet/puppetdb            "dumb-init /docker-e…"   31 minutes ago      Up 31 minutes               0.0.0.0:32778->8080/tcp, 0.0.0.0:32777->8081/tcp   singlehost_puppetdb_1
ffe0ada7cb71        puppet/puppetexplorer      "/usr/bin/caddy"         31 minutes ago      Up 31 minutes               0.0.0.0:80->80/tcp                                 singlehost_puppetexplorer_1
1555dbfdc6a2        puppet/puppetboard         "/bin/sh -c '/usr/bi…"   31 minutes ago      Up 31 minutes (unhealthy)   0.0.0.0:8000->8000/tcp                             singlehost_puppetboard_1
9d70a90721d4        puppet/puppetdb-postgres   "docker-entrypoint.s…"   31 minutes ago      Up 31 minutes               5432/tcp                                           postgres
```

Okay, so port 8140 on the `puppet/puppetserver` image was mapped to port 32779 on my host.  So I just needed to modify my puppet.conf to set that.

I created the file `C:\ProgramData\PuppetLabs\puppet\etc\puppet.conf` with the contents

``` ini
[main]
masterport=32779
```

And I ran puppet agent again

```powershell
PS> puppet agent -t --noop
Info: Caching certificate for ca
Info: csr_attributes file loading from C:/ProgramData/PuppetLabs/puppet/etc/csr_attributes.yaml
Info: Creating a new SSL certificate request for win-hbiod5i9gso.gallifrey.local
Info: Certificate Request fingerprint (SHA256): 31:E3:C8:63:7A:75:81:2F:B4:37:74:2A:3D:CE:31:1A:9E:27:70:19:25:3C:80:C8:BF:77:0E:E0:1C:F8:67:86
Info: Caching certificate for win-hbiod5i9gso.gallifrey.local
Info: Caching certificate_revocation_list for ca
Info: Caching certificate for ca
Info: Using configured environment 'production'
Info: Retrieving pluginfacts
Info: Retrieving plugin
Info: Retrieving locales
Info: Applying configuration version '1524633937'
Info: Creating state file C:/ProgramData/PuppetLabs/puppet/cache/state/state.yaml
Notice: Applied catalog in 0.09 seconds
```

Success! and now Puppet Board was showing something useful

![Puppet Board with a node]({{ site.url }}/blog-images/puppet-docker-003.png){: .align-center}

# Running Puppet Code

So now I had a master running on Windows (albeit on Docker on Windows) with a Windows agent connected.  Now it was time to do something useful.  The compose file exports the Puppet Server code directory as `./code` which means I could just add files and folders on my local computer in that directory, and they would be seen by Puppet Server.

I created the standard directory structure of a production environment and installed a simple windows module, [Windows Disk Facts](https://forge.puppet.com/dylanratcliffe/windows_disk_facts) from Dylan Ratcliffe (A fellow Aussie!)

``` powershell
PS> md code\environments\production\modules


    Directory: C:\puppetstack\code\environments\production


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
d-----       25/04/2018   2:20 PM                modules


PS> cd code\environments\production\modules

PS> puppet module install dylanratcliffe-windows_disk_facts --modulepath .
Notice: Preparing to install into C:/puppetstack/code/environments/production/modules ...
Notice: Downloading from https://forgeapi.puppet.com ...
Notice: Installing -- do not interrupt ...
C:/puppetstack/code/environments/production/modules
└── dylanratcliffe-windows_disk_facts (v0.2.1)
```

And now I went back to my Windows VM and ran puppet agent again

``` powershell
PS> puppet agent -t
Info: Using configured environment 'production'
Info: Retrieving pluginfacts
Info: Retrieving plugin
Notice: /File[C:/ProgramData/PuppetLabs/puppet/cache/lib/facter]/ensure: created
Notice: /File[C:/ProgramData/PuppetLabs/puppet/cache/lib/facter/windows_disks.rb]/ensure: defined content as '{md5}8b76765771fe4614ed09553aadfb8d1c'
Notice: /File[C:/ProgramData/PuppetLabs/puppet/cache/lib/facter/windows_drives.rb]/ensure: defined content as '{md5}35868d5da45430664f2c5581a5751658'
Notice: /File[C:/ProgramData/PuppetLabs/puppet/cache/lib/facter/windows_partitions.rb]/ensure: defined content as '{md5}532fbef4aa15a16e2ed52ec6578dcf2c'
Notice: /File[C:/ProgramData/PuppetLabs/puppet/cache/lib/puppet_x]/ensure: created
Notice: /File[C:/ProgramData/PuppetLabs/puppet/cache/lib/puppet_x/disk_facts]/ensure: created
Notice: /File[C:/ProgramData/PuppetLabs/puppet/cache/lib/puppet_x/disk_facts/underscore.rb]/ensure: defined content as '{md5}2c50d96e067ec3c02560e3630ac6aa87'
Info: Retrieving locales
Info: Loading facts
Info: Caching catalog for win-hbiod5i9gso.gallifrey.local
Info: Applying configuration version '1524637450'
Notice: Applied catalog in 0.04 seconds
```

We can see that it downloaded some new fact files which came from the windows_disk_facts module.  So now this information should be available to view in Puppet Board

![Puppet Board with a node]({{ site.url }}/blog-images/puppet-docker-004.png){: .align-center}

And sure enough if I browse the facts on my Windows agent, I can see the new fact `drives` is populated with;

``` json
{
  "C": {
    "description": "",
    "used_bytes": 19989024768,
    "display_root": "",
    "free_bytes": 43880783872,
    "drivetype": "Fixed",
    "root": "C:\\",
    "name": "C"
  }
}
```

# Wrapping up

So I now had a working Puppet Master setup running in Docker on Windows, with a connected Windows agent, and applying a standard puppet code layout which I could edit on my host computer and then test on the Windows agent.

And I could quickly create and destroy this environment as needed, on my development laptop without any need for internet connection!

The docker-compose files are available in my [blog code repository](https://github.com/glennsarti/code-glennsarti.github.io/tree/master/puppet-in-docker).

# Improvements

There are still a number of improvements that could be made;

## Use static port for Puppet Server

Instead of getting docker to dynamically assign a port for the Puppet Server, just statically assign it to 8140 by changing the ports to `8140:8140`;

``` yaml
services:
  puppet:
    ...
    ports:
      - "8140:8140"
```

## Use the load balancer but with TCP endpoint

I removed the load balancer for my needs, however gobetween does support TCP docker endpoints, as well as Unix sockets.

`docker_endpoint = "unix:///var/run/docker.sock"` vs `docker_endpoint = "http://localhost:2375"`

## Use a Windows container for the agent

Now that we have Linux Containers on Windows (LCOW) it may be possible to add Windows Server Core container with Puppet Agent installed.  Which means I won't need to manually spin up a Windows VM for testing.
