---
title: Using SCCM as a Puppet ENC - Part 3
excerpt: Configuring Puppet to use an ENC compatible Web Service
category:
  - blog
header:
  overlay_image: header-puppet-console-dark.png
  teaser: teaser-puppet-sccm.png
tags:
  - puppet
  - enc
  - powershell
  - sccm
  - configuration manager
---

In [Part 1]({{ site.url }}/blog/puppet-enc-sccm-part1) of this series we created the required Roles, Profiles and Class as collections within SCCM.

In [Part 2]({{ site.url }}/blog/puppet-enc-sccm-part2) we created a web service which queries SCCM and then creates ENC compatible responses, which can then be consumed by Puppet.

In this next post, we'll configure Puppet to use our ENC web service and then try out some node classifications for real.

All scripts and examples are at the [blog code repository](https://github.com/glennsarti/code-glennsarti.github.io/tree/master/puppet-enc-sccm/)


## Setting up a Puppet Master and Agent

If you already have a running Puppet master and agent, you can skip this section.  
{: .notice}

Puppet offers a Learning Virtual Machine (VM) which is a quick way to create Puppet Master.

Download and import the Virtual Machine into your choice of virtualisation application (Hyper-V, Fusion, Virtual Box) [Instructions](https://puppet.com/download-learning-vm){: .btn .btn--info}

Once you have a Puppet master running, install an Puppet agent onto a separate computer and add the agent to the Puppet Master. [Instructions](https://docs.puppet.com/pe/latest/install_agents.html){: .btn .btn--info}


## Start the ENC Web Service

Using the instructions in [Part 2]({{ site.url }}/blog/puppet-enc-sccm-part2) start the ENC Web Service.  Make a note of the Hostname or IP Address of this computer as you'll need it later.  In this example the ENC Web Service is `172.16.68.145`

Try connecting to the Web Service using powershell.  We can use a fake hostname for the time being:

```s
PS C:\Users\Administrator> (invoke-webrequest -uri 'http://localhost:172.16.68.145/hostname').RawContent
HTTP/1.1 200 OK
Content-Length: 0
Date: Sat, 02 Jul 2016 23:21:43 GMT
Server: Microsoft-HTTPAPI/2.0
```

## Testing the ENC Web Serivce on the Puppet Master

Now that the ENC Web Service is running we can try connecting from the Puppet Master:

* Either SSH into the Puppet Master Virtual Machine, or use the console.

* Use `curl` to send a web request.  The `--verbose` argument shows the HTTP response headers.

```s
root@learning:~ # curl http://172.16.68.145:8080/hostname --verbose
* About to connect() to 172.16.68.145 port 8080 (#0)
*   Trying 172.16.68.145...
* Connected to 172.16.68.145 (172.16.68.145) port 8080 (#0)
> GET /hostname HTTP/1.1
> User-Agent: curl/7.29.0
> Host: 172.16.68.145:8080
> Accept: */*
>
< HTTP/1.1 200 OK
< Content-Length: 0
< Server: Microsoft-HTTPAPI/2.0
< Date: Sat, 02 Jul 2016 23:27:12 GMT
<
* Connection #0 to host 172.16.68.145 left intact
root@learning:~ #
```

## Configure Puppet Master

### Create a shell script

Puppet can not call curl directly so we need a simple shell script that Puppet can call.

* Create a file called `/usr/local/bin/sccm_enc.sh`

``` bash
#! /bin/sh

computername="$1"

curl -k "http://172.16.68.145:8080/${computername}"
```

* Change the permissions

``` bash
chmod 755 /usr/local/bin/sccm_enc.sh
```

Let's try out the script:

```s
root@learning:~ # /usr/local/bin/sccm_enc.sh hostname
root@learning:~ #
```

Hrmm ... not very interesting.  Let's use a real hostname, like we did in [Part 2]({{ site.url }}/blog/puppet-enc-sccm-part2):

```s
root@learning:~ # /usr/local/bin/sccm_enc.sh WINDOWS001.sccm-demo.local
---
classes:
    shared::iis::no_default_website:
    advworks::website:
    shared::iis:
    shared::iis::security:
environment: production
root@learning:~ #
```

Much better!


### Set Puppet Master settings

The [Puppet documentation](https://docs.puppet.com/guides/external_nodes.html#connecting-an-enc) says in order to use an ENC, we need to define two settings on the Puppet master; `node_terminus` and `external_nodes`.  By default they are set to `classifier` and `none`.  We can use the commands below to get the current settings:

```s
root@learning:~ # puppet config print --section master node_terminus
classifier
root@learning:~ # puppet config print --section master external_nodes
none
root@learning:~ #
```

Now to set the ENC Web Service:

```s
root@learning:~ # puppet config set --section master node_terminus exec
root@learning:~ # puppet config set --section master external_nodes /usr/local/bin/sccm_enc.sh
root@learning:~ #
```

These settings require the Puppet Master to restart.  The simplest method is just to restart the Learning VM with `sudo shutdown -r now`.  Once the Puppet Master is back online we can SSH in and verify our changes are in effect:

```s
root@learning:~ # puppet config print --section master node_terminus
exec
root@learning:~ # puppet config print --section master external_nodes
/usr/local/bin/sccm_enc.sh
root@learning:~ #
```

So let's do a Puppet run on the master to see if anything has broken.  We can use the [`puppet agent -t`](https://docs.puppet.com/puppet/4.5/reference/man/agent.html) command for this:

```s
root@learning:~ # puppet agent -t
Warning: Unable to fetch my node definition, but the agent run will continue:
Warning: Find /puppet/v3/node/learning.puppetlabs.vm?environment=production&transaction_uuid=39901b67-e431-416b... resulted in 404 with the message: Not Found: Could not find node learning.puppetlabs.vm
Info: Retrieving pluginfacts
Info: Retrieving plugin
Info: Loading facts
Error: Could not retrieve catalog from remote server: Error 400 on SERVER: Could not find node 'learning.puppetlabs.vm'; cannot compile
Warning: Not using cache on failed catalog
Error: Could not retrieve catalog; skipping run
root@learning:~ # 
```

Oh, that's not good.  The Puppet master no longer has a node definition. Looking at the Web Service output we can see why:

```
VERBOSE: Listening at http://*:8080/...
VERBOSE: Request: NodeName = learning.puppetlabs.vm
VERBOSE: Querying learning.puppetlabs.vm ...
VERBOSE: Unable to find any environments
VERBOSE: Response:
```

We haven't defined a Role or Environment for our Puppet Master!  Fortunately the Puppet Console still holds all of the node classification information, so we can use that to build the required response for the Web Service.

Open the Puppet Console and select `Nodes`, and then `Inventory` under `Node Management`.  Click on the Puppet Master server name, in our example it will be `learning.puppet.vm`.  This will display the information about the Puppet Master.  Click on the `Classes` tab and it will display all of the classes assigned to the Puppet Master.  This is information needed for our ENC.  An example class list is shown below;

![Puppet Master classes]({{ site.url }}/blog-images/sccm-puppet-enc-009.png)

Using this list of classes we can generate an example YAML node definitition for the master, assume the Puppet environment is `production`:

```yaml
---
classes:
    pe_console_prune:
    pe_repo:
    pe_repo::platform::el_7_x86_64:
    puppet_enterprise:
    puppet_enterprise::license:
    puppet_enterprise::profile::agent:
    puppet_enterprise::profile::amq::broker:
    puppet_enterprise::profile::certificate_authority:
    puppet_enterprise::profile::console:
    puppet_enterprise::profile::master:
    puppet_enterprise::profile::master::mcollective:
    puppet_enterprise::profile::mcollective::agent:
    puppet_enterprise::profile::mcollective::peadmin:
    puppet_enterprise::profile::orchestrator:
    puppet_enterprise::profile::puppetdb:
environment: production
```

However this isn't the complete definition as there are normally some class parameters created by default.  To find them all is a bit labourious.  Open each `Source Group` and then click on the `Classes` tab.  In the Learning VM there are two Source Groups which have parameters; `PE Console` and `PE Infrastructure`.  Here's what the `PE Infrastructure` classes look like;

![PE infrastructure parameters]({{ site.url }}/blog-images/sccm-puppet-enc-010.png)

Combining these parameters we can derive the full node definition:

```yaml
---
classes:
    pe_console_prune:
      prune_upto: 30
    pe_repo:
    pe_repo::platform::el_7_x86_64:
    puppet_enterprise:
      mcollective_middleware_hosts:
        - learning.puppetlabs.vm
      use_application_services: true
      database_host: learning.puppetlabs.vm
      puppetdb_host: learning.puppetlabs.vm
      database_port: 5432
      database_ssl: true
      puppet_master_host: learning.puppetlabs.vm
      certificate_authority_host: learning.puppetlabs.vm
      console_port: 443
      puppetdb_database_name: pe-puppetdb
      puppetdb_database_user: pe-puppetdb
      pcp_broker_host: learning.puppetlabs.vm
      puppetdb_port: 8081
      console_host: learning.puppetlabs.vm
    puppet_enterprise::license:
    puppet_enterprise::profile::agent:
    puppet_enterprise::profile::amq::broker:
    puppet_enterprise::profile::certificate_authority:
    puppet_enterprise::profile::console:
    puppet_enterprise::profile::master:
    puppet_enterprise::profile::master::mcollective:
    puppet_enterprise::profile::mcollective::agent:
    puppet_enterprise::profile::mcollective::peadmin:
    puppet_enterprise::profile::orchestrator:
    puppet_enterprise::profile::puppetdb:
environment: production
```

## Updating the Web Service for static definitions

Now that we know we need to add a definition for the Puppet Master to the ENC, how do we do this?  There a couple of ways of doing it;

* Update the `/usr/local/bin/sccm_enc.sh` script

We could update the script so that if our Web Service fails to find a definition then it could query the original [Node Classifier API](https://docs.puppet.com/pe/latest/nc_index.html).  This is the best of both worlds where we can use the the native capabilites of Puppet to manage itself, and then use SCCM for other Nodes.

However this a little advanced for this blog post.  In Part 4 of this blog series we can revisit this.

* Add all of the nodes and classes to SCCM

This would be fairly straight forward to add into SCCM but does require a bit work to get the system objects imported into SCCM.  Getting non-Windows computers to register in SCCM is quite tricky.  Also our ENC doesn't support class parameters yet.  In Part 4 of this blog series we can revisit this.

* Update the ENC Web Service and use a simple static responder

In this case, when a request for a specific hostname is received, we can return a static response instead of querying the SCCM database.  We'll use this method in this post as it's very simple to implement.

### ENC Web Service with static definitions

Source Code - [SCCM-ENC-WebService-WithStatic.ps1](https://github.com/glennsarti/code-glennsarti.github.io/blob/master/puppet-enc-sccm/SCCM-ENC-WebService-WithStatic.ps1)

A new function called `Get-StaticNodeResponse` takes a single input parameter of the node hostname, and then uses a simple `switch` statement to either return a YAML node classification string, or an empty string.

```powershell
function Get-StaticNodeResponse($nodeName) {
  switch ($nodeName) {
    'learning.puppetlabs.vm' {
      return (@"
---
classes:
    pe_console_prune:
      prune_upto: 30
    pe_repo:
...
...
...
environment: production
"@)
    }
    default { return '' }
  }
}
```

The `Get-NodeResponse` function is modified to first call the `Get-StaticNodeResponse` function.  If the static response is not empty the it returns the static response, otherwise it continues the usual process of querying SCCM.

``` powershell
function Get-NodeResponse($NodeName) {
  try
  {
    # Check for static definition
    $response = [string](Get-StaticNodeResponse -nodeName $NodeName)
    if ($response -ne '') {
      Write-Verbose "Using static response"
      return $response
    }
...
```



### Try out the new ENC Web Service

Again, we run `puppet agent -t`:

```s
root@learning:~ # puppet agent -t
Info: Using configured environment 'production'
Info: Retrieving pluginfacts
Info: Retrieving plugin
Info: Loading facts
Info: Caching catalog for learning.puppetlabs.vm
Info: Applying configuration version '1467606433'
Notice: /Stage[main]/Puppet_enterprise::Profile::Master::Classifier/Pe_ini_setting[node_terminus]/value: value changed 'exec' to 'classifier'
Info: /Stage[main]/Puppet_enterprise::Profile::Master::Classifier/Pe_ini_setting[node_terminus]: Scheduling refresh of Service[pe-puppetserver]
Info: Class[Puppet_enterprise::Profile::Master::Classifier]: Scheduling refresh of Service[pe-puppetserver]
Notice: /Stage[main]/Puppet_enterprise::Master::Puppetserver/Service[pe-puppetserver]: Triggered 'refresh' from 2 events
Notice: Applied catalog in 48.59 seconds
root@learning:~ #
```

Uh oh.  Looks like the `node_terminus` setting was put back to `classifier`.

Unfortunately there is no configuration option for this setting but we can disable the resource being applied by setting the `classifier host` class parameter to an empty string.  Inside the `Get-StaticNodeResponse` function we can update the static classification script from:

``` yaml
...
    puppet_enterprise::profile::master:
    puppet_enterprise::profile::master::mcollective:
...
```

to

``` yaml
...
    puppet_enterprise::profile::master:
      classifier_host: ""
    puppet_enterprise::profile::master::mcollective:
...
```

We then have to run the commands below to update the Node Classifier, again, and then reboot the Puppet Master VM:

```s
root@learning:~ # puppet config set --section master node_terminus exec
root@learning:~ # puppet config set --section master external_nodes /usr/local/bin/sccm_enc.sh
root@learning:~ # sudo shutdown -r now
```

Once the Puppet Master has finished restarting, log into the VM and then run `puppet agent -t`:

```s
root@learning:~ # puppet agent -t
Info: Using configured environment 'production'
Info: Retrieving pluginfacts
Info: Retrieving plugin
Info: Loading facts
Info: Caching catalog for learning.puppetlabs.vm
Info: Applying configuration version '1467848637'
Notice: Applied catalog in 9.66 seconds
root@learning:~ #
```

Success! The Puppet Master has a definition and found nothing to update.  Also the ENC Web Service logs show it was queried and the expected classes are returned:

```s
VERBOSE: Listening at http://*:8080/...
VERBOSE: Request: NodeName = learning.puppetlabs.vm
VERBOSE: Using static response
VERBOSE: Response: ---
classes:
    pe_console_prune:
      prune_upto: 30
    pe_repo:
    pe_repo::platform::el_7_x86_64:
    puppet_enterprise:
      mcollective_middleware_hosts:
        - learning.puppetlabs.vm
...
```

Now that the Puppet Master is functioning correctly we can move on to classifying other nodes.

## Classifying Nodes in SCCM

Now that the Puppet Master is functioning correctly, we can now start classifying nodes in SCCM and see the results on an agent.  The test node in the following examples a Windows 10 desktop with a name of `desktop-urvo7t4.sccm-demo.local`.  It's not your typical server setup, but it will serve the purposes of testing an ENC.

We should already have the Puppet Agent installed and registered with the Puppet Master.  Let's see what happens when we do a Puppet run with `puppet agent -t`:

```
PS C:\Programdata\PuppetLabs\puppet> puppet agent -t
Warning: Unable to fetch my node definition, but the agent run will continue:
Warning: Find /puppet/v3/node/desktop-urvo7t4.sccm-demo.local?environment=production&transaction_uuid=60db063d-52f1... resulted in 404 with the message: Not Found: Could not find node desktop-urvo7t4.sccm-demo.local
Info: Retrieving pluginfacts
Info: Retrieving plugin
Info: Loading facts
Error: Could not retrieve catalog from remote server: Error 400 on SERVER: Could not find node 'desktop-urvo7t4.sccm-demo.local'; cannot compile
Warning: Not using cache on failed catalog
Error: Could not retrieve catalog; skipping run
PS C:\Programdata\PuppetLabs\puppet>
```

Well, that's kind of expected as we haven't added this computer in SCCM yet.  Install the SCCM Agent and it should soon appear in the SCCM Console as a Device.

Remember in Part 1 we created all of those collections? Well now it's time to use them.  All nodes need to have an environment set so first we add the test node to the `Puppet::environment::production` collection in SCCM.

In these examples I used Direct Membership, but this will work fine with the other types of [collection membership](https://technet.microsoft.com/en-us/library/gg712295.aspx)
{: .notice}

Running `puppet agent -t` on the test node shows the following

```
PS C:\Programdata\PuppetLabs\puppet> puppet agent -t
Info: Using configured environment 'production'
Info: Retrieving pluginfacts
Info: Retrieving plugin
Info: Loading facts
Info: Caching catalog for desktop-urvo7t4.sccm-demo.local
Info: Applying configuration version '1468202956'
Notice: Applied catalog in 0.18 seconds
PS C:\Programdata\PuppetLabs\puppet>
```

And the corresponding output of the Web Service

```
VERBOSE: Request: NodeName = desktop-urvo7t4.sccm-demo.local
VERBOSE: Querying desktop-urvo7t4.sccm-demo.local ...
VERBOSE: Found Environment collection Puppet::Environment::production
VERBOSE: Response: ---
classes:
environment: production
```

### AdvWorks-Webserver Role

Excellent! The test node succesfully had a node classification and is in the `production` environment as expected.  Now it's time to do something more useful and assign a role.  We then add the test node to the Web Server Role collection, which is called `Puppet::Role::AdvWorks-WebServer`, and then run Puppet:

```s
PS C:\Programdata\PuppetLabs\puppet> puppet agent -t
Info: Using configured environment 'production'
Info: Retrieving pluginfacts
Info: Retrieving plugin
Info: Loading facts
Error: Could not retrieve catalog from remote server: Error 400 on SERVER: Could not find class shared::iis::no_default_website for desktop-urvo7t4.sccm-demo.local on node desktop-urvo7t4.sccm-demo.local
Warning: Not using cache on failed catalog
Error: Could not retrieve catalog; skipping run
PS C:\Programdata\PuppetLabs\puppet>
```

And the corresponding output of the Web Service

```s
VERBOSE: Request: NodeName = desktop-urvo7t4.sccm-demo.local
VERBOSE: Querying desktop-urvo7t4.sccm-demo.local ...
VERBOSE: Found Environment collection Puppet::Environment::production
VERBOSE: Found Role collection Puppet::Role::AdvWorks-WebServer
VERBOSE: Found Profile collection Puppet::Profile::IISWebServer
VERBOSE: Found Profile collection Puppet::Profile::IISBaselineSecurity
VERBOSE: Found Profile collection Puppet::Profile::AdvWorksWebsite
VERBOSE: Found Class collection Puppet::Class::shared::iis
VERBOSE: Found Class collection Puppet::Class::shared::iis::no_default_website
VERBOSE: Found Class collection Puppet::Class::shared::iis::security
VERBOSE: Found Class collection Puppet::Class::advworks::website
VERBOSE: Response: ---
classes:
    shared::iis::no_default_website:
    advworks::website:
    shared::iis:
    shared::iis::security:
environment: production
```

This isn't quite what we expected.  The Web Service output shows it retrieved the correct classes for the Web Server role, but the Puppet run failed.  After a quick look on the Puppet Master, the answer is obvious.  We hadn't yet added the modules for the Adventure Works site onto the Puppet Master.  This meant that even though the node classified correctly, the Puppet Master couldn't compile a catalog for the node.

I created demo modules and are available in the [blog code repository](https://github.com/glennsarti/code-glennsarti.github.io/tree/master/advworks-puppet-modules).  These modules merely output Notify messages on the console for the various class names.  On the Puppet Learning VM they can be copied to `/etc/puppetlabs/code/environment/production/modules`.

Once the modules are in place we can do another puppet run:

```s
PS C:\Programdata\PuppetLabs\puppet> puppet agent -t
Info: Using configured environment 'production'
Info: Retrieving pluginfacts
Info: Retrieving plugin
Info: Loading facts
Info: Caching catalog for desktop-urvo7t4.sccm-demo.local
Info: Applying configuration version '1468195467'
Notice: shared::iis::no_default_website class
Notice: /Stage[main]/Shared::Iis::No_default_website/Notify[shared::iis::no_default_website class]/message: defined 'message' as 'shared::iis::no_default_website class'
Notice: advworks::website class
Notice: /Stage[main]/Advworks::Website/Notify[advworks::website class]/message: defined 'message' as 'advworks::website class'
Notice: shared::iis class
Notice: /Stage[main]/Shared::Iis/Notify[shared::iis class]/message: defined 'message' as 'shared::iis class'
Notice: shared::iis::security class
Notice: /Stage[main]/Shared::Iis::Security/Notify[shared::iis::security class]/message: defined 'message' as 'shared::iis::security class'
Notice: Applied catalog in 0.20 seconds
PS C:\Programdata\PuppetLabs\puppet>
```

Success!  The node is classified correctly and we can see that the correct classes have put out Notify messages in the Puppet console.

### AdvWorks-Database Role

So let's now make the node a Database Server.  We add the node to the Database Role, which has a collection called `Puppet::Role::AdvWorks-Database`.  We then do a puppet run:

```s
PS C:\Programdata\PuppetLabs\puppet> puppet agent -t
Warning: Unable to fetch my node definition, but the agent run will continue:
Warning: Find /puppet/v3/node/desktop-urvo7t4.sccm-demo.local?environment=production&transaction_uuid=75033dac-... resulted in 404 with the message: Not Found: Could not find node desktop-urvo7t4.sccm-demo.local
Info: Retrieving pluginfacts
Info: Retrieving plugin
Info: Loading facts
Error: Could not retrieve catalog from remote server: Error 400 on SERVER: Could not find node 'desktop-urvo7t4.sccm-demo.local'; cannot compile
Warning: Not using cache on failed catalog
Error: Could not retrieve catalog; skipping run
PS C:\Programdata\PuppetLabs\puppet>
```

And the corresponding output of the Web Service

```s
VERBOSE: Request: NodeName = desktop-urvo7t4.sccm-demo.local
VERBOSE: Querying desktop-urvo7t4.sccm-demo.local ...
VERBOSE: Found Environment collection Puppet::Environment::production
VERBOSE: Found Role collection Puppet::Role::AdvWorks-WebServer
VERBOSE: Found Role collection Puppet::Role::AdvWorks-Database
VERBOSE: Found Profile collection Puppet::Profile::IISWebServer
VERBOSE: Found Profile collection Puppet::Profile::IISBaselineSecurity
VERBOSE: Found Profile collection Puppet::Profile::AdvWorksWebsite
VERBOSE: Found Profile collection Puppet::Profile::MSSQLServer
VERBOSE: Found Profile collection Puppet::Profile::MSSQLServerBaselineSecurity
VERBOSE: Found Profile collection Puppet::Profile::AdvWorksDatabase
VERBOSE: Found Class collection Puppet::Class::shared::iis
VERBOSE: Found Class collection Puppet::Class::shared::iis::no_default_website
VERBOSE: Found Class collection Puppet::Class::shared::iis::security
VERBOSE: Found Class collection Puppet::Class::advworks::website
VERBOSE: Found Class collection Puppet::Class::shared::mssql
VERBOSE: Found Class collection Puppet::Class::shared::mssql::security
VERBOSE: Found Class collection Puppet::Class::advworks::database
VERBOSE: Node is a member of more than one role
VERBOSE: Response:
```

Whoops. We forgot that we can **only assign one** Role to a node.  Remove the test node from the `Puppet::Role::AdvWorks-WebServer` collection and try again:

```s
PS C:\Programdata\PuppetLabs\puppet> puppet agent -t
Info: Using configured environment 'production'
Info: Retrieving pluginfacts
Info: Retrieving plugin
Info: Loading facts
Info: Caching catalog for desktop-urvo7t4.sccm-demo.local
Info: Applying configuration version '1468203866'
Notice: shared::mssql class
Notice: /Stage[main]/Shared::Mssql/Notify[shared::mssql class]/message: defined 'message' as 'shared::mssql class'
Notice: shared::mssql::security class
Notice: /Stage[main]/Shared::Mssql::Security/Notify[shared::mssql::security class]/message: defined 'message' as 'shared::mssql::security class'
Notice: advworks::database class
Notice: /Stage[main]/Advworks::Database/Notify[advworks::database class]/message: defined 'message' as 'advworks::database class'
Notice: Applied catalog in 0.20 seconds
PS C:\Programdata\PuppetLabs\puppet>
```

And the corresponding output of the Web Service

```s
VERBOSE: Request: NodeName = desktop-urvo7t4.sccm-demo.local
VERBOSE: Querying desktop-urvo7t4.sccm-demo.local ...
VERBOSE: Found Environment collection Puppet::Environment::production
VERBOSE: Found Role collection Puppet::Role::AdvWorks-Database
VERBOSE: Found Profile collection Puppet::Profile::MSSQLServer
VERBOSE: Found Profile collection Puppet::Profile::MSSQLServerBaselineSecurity
VERBOSE: Found Profile collection Puppet::Profile::AdvWorksDatabase
VERBOSE: Found Class collection Puppet::Class::shared::mssql
VERBOSE: Found Class collection Puppet::Class::shared::mssql::security
VERBOSE: Found Class collection Puppet::Class::advworks::database
VERBOSE: Response: ---
classes:
    shared::mssql:
    shared::mssql::security:
    advworks::database:
environment: production
```

As expected, the node classification picks up the database classes.

### AdvWorks-LoadBalancer Role

And finally, what about acting as a Load Balancer.  We add the Load Balancer Role which has SCCM collection called `Puppet::Role::AdvWorks-Load-Balancer` and remove it from the Database Role collection.  We then do a Puppet run:

```s
PS C:\Programdata\PuppetLabs\puppet> puppet agent -t
Info: Using configured environment 'production'
Info: Retrieving pluginfacts
Info: Retrieving plugin
Info: Loading facts
Info: Caching catalog for desktop-urvo7t4.sccm-demo.local
Info: Applying configuration version '1468204005'
Notice: haproxy class
Notice: /Stage[main]/Haproxy/Notify[haproxy class]/message: defined 'message' as 'haproxy class'
Notice: Applied catalog in 0.18 seconds
PS C:\Programdata\PuppetLabs\puppet>
```

And the corresponding output of the Web Service

```s
VERBOSE: Request: NodeName = desktop-urvo7t4.sccm-demo.local
VERBOSE: Querying desktop-urvo7t4.sccm-demo.local ...
VERBOSE: Found Environment collection Puppet::Environment::production
VERBOSE: Found Role collection Puppet::Role::AdvWorks-Load-Balancer
VERBOSE: Found Profile collection Puppet::Profile::HAProxyService
VERBOSE: Found Class collection Puppet::Class::haproxy
VERBOSE: Response: ---
classes:
    haproxy:
environment: production
```

And again as expected, the node classification picks up the HA Proxy classes.


## Wrapping up

So we've managed to create the Roles and Profiles pattern in SCCM, create a Web Service which serves ENC compatible YAML files and configure Puppet to use SCCM (with Static classifications) to classify Nodes!

Many people feel that SCCM and Puppet are mutually exclusive, in that you can't use both at the same time, but I like to think that they are complimentary. SCCM does a lot of things Puppet does not, Puppet does a lot of things that SCCM does not.  If you need to, you can integrate the two systems together to manage your server fleet.

## What's next?

There are still two outstanding issues to look at regarding the Puppet Master node definitions.  In Part 4 we'll revisit the ENC script and look at using the inbuilt Puppet Node Classifier as a fallback. We'll also look at adding class parameters in SCCM and exposing them via the ENC.
