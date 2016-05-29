---
title: Using SCCM as a Puppet ENC - Part 1
excerpt: Creating the Puppet Roles and Profiles schema in Microsoft System Center Configuration Manager
category:
  - blog
tags:
  - puppet
  - powershell
  - sccm
  - configuration manager
---

System Center Configuration Manager (SCCM) is a common tool used by Windows administrators to manage their servers and desktops.  While Puppet does compete with SCCM in some areas, they can be used together.  Puppet can use other sources to assign classes to nodes using an [External Node Classifier](https://docs.puppet.com/guides/external_nodes.html)

> An external node classifier is an arbitrary script or application which can tell Puppet which classes a node should have. It can replace or work in concert with the node definitions in the main site manifest (site.pp).
> 
> Depending on the external data sources you use in your infrastructure, building an external node classifier can be a valuable way to extend Puppet.

In this multipart series we'll create an ENC which uses SCCM for its information.


## What are Puppet Roles and Profiles?

A common pattern used in Puppet is to use Roles and Profiles to abstract which classes are assigned to particular nodes.  Here are some links about the Roles and Profiles pattern from people who can explain it much better than I can;

[Assigning configuration data with role and profile modules](https://docs.puppet.com/pe/latest/puppet_assign_configurations.html#assigning-configuration-data-with-role-and-profile-modules)

[Designing Puppet – Roles and Profiles](http://www.craigdunn.org/2012/05/239/)

[Roles and Profiles](http://puppetlunch.com/puppet/roles-and-profiles.html)

[Intro to roles and profiles with Puppet and Hiera](https://rnelson0.com/2014/07/14/intro-to-roles-and-profiles-with-puppet-and-hiera/)


## How do we express this in SCCM?

The rules for Roles and Profiles can be summarised as;

* A Node can be assigned to one, and only one, Role

* A Role may be assigned to zero or more Profiles

* A Profile may be assigned to zero or more Classes

Using the information in the ENC documentation, an additional rule can be added;

* A Node can only be assigned to one, and only one, environment

![Nodes and Profiles Diagram]({{ site.url }}/blog-images/sccm-puppet-enc-002.png)


## Putting it into practice

So all of this theory is nice but how do we actually put this into practice.  Let's use an example.

> The Adventure Works website is a simple three tier application
>
> A HTTP Load Balancer that reverse proxies connections to Web Servers.  The Web Servers connect to a Database.

All scripts and examples are at the [blog code repository](https://github.com/glennsarti/code-glennsarti.github.io/tree/master/puppet-enc-sccm/)


### Roles and Profiles

For the Adventure Works website we need three roles;

- A load balancer role; `AdvWorks-Load-Balancer`

- A web server role; `AdvWorks-WebServer`

- A database role; `AdvWorks-Database`


Unlike Roles, Profiles are technology focused and we can make them resuable for other applications, not just Adventure Works

- A HAProxy profile; `HAProxyService`
  
  This profile would install and configure HA Proxy

- Basic IIS and MS SQL Database profiles; `IISWebServer` and `MSSQLServer`
  
  These reusable profiles will install IIS and MSSQL
  
- Security baselines for IIS and MS SQL Database profiles; `IISBaselineSecurity` and `MSSQLServerBaselineSecurity`

  These reuable profiles will enforce basic security settings for IIS and MS SQL
  
- Adventure Works profiles for IIS and MS SQL database; `AdvWorksWebsite` and `AdvWorksDatabase`

  These profiles will install the Adventure Works website and database


Classes are implementions of the Profiles and are generally tied to Puppet Modules.  For the sake of the Adventure Works example, we'll make up some Modules that the Profiles will use;

``` puppet
modules/
  +- advworks/
  |   +- manifests/
  |         +- website.pp
  |         +- database.pp
  +- haproxy/
  |    ...
  +- shared/
  |    +- manifests/
  |         +- iis.pp
  |         +- iis/
  |         |    +- no_default_website.pp
  |         |    +- security.pp
  |         +- mssql.pp
  |         +- mssql/
  |              +- security.pp
  +- windowsfeatures/
       ...
```

Given the basic module layout above, we can associate Profiles to Classes;

```
HAProxyService
  - haproxy

IISWebServer
  - shared::iis
  - shared::iis::no_default_website

IISBaselineSecurity
  - shared::iis::security
  
AdvWorksWebsite
  - advworks::website

MSSQLServer
  - shared::mssql
      
MSSQLServerBaselineSecurity
  - shared::mssql::security

AdvWorksDatabase
  - advworks::database
```


### Using collections for the roles and profiles

So now that we know the Roles-Profiles-Classes structure, it can be created using collections in SCCM;

- Each Environment, Role, Profile and Class has its own collection.  Collection membership is used to express the relationships between the collections

- System resources (Nodes) are members of Role collections.  Direct or query based membership could be used.

- Profile collections are assigned to Roles using collection membership via Include Collections

- Class collections are assigned to Profiles using collection membership also via Include Collections

It should noted that there's no concept of unique membership in SCCM, so it is possible that a Node could have two roles.  Instead of restricting this in SCCM, we can push this check to the ENC code.


### Creating collection folders

Firstly, to make things easier we'll created some folders in the Device Collections

![Device Collection Hierarchy]({{ site.url }}/blog-images/sccm-puppet-enc-001.jpg)


### Creating the collection hierarchy

SCCM has some good powershell integration so let's use powershell to create the collections and memberships.  This script lacks idempotency, but it gets the job done.


#### Create-SCCM-Hierarchy.ps1

[Source](https://github.com/glennsarti/code-glennsarti.github.io/blob/master/puppet-enc-sccm/Create-SCCM-Hierarchy.ps1)

``` powershell
# The SCCM Module is not in the usual Autoload location
Import-Module 'C:\Program Files (x86)\Microsoft Configuration Manager\AdminConsole\bin\ConfigurationManager.psd1'

# Set the location to the SCCM Site of this server
$sccmSite = (Get-PSDrive | ? { $_.Provider -like '*CMSite'} | Select -First 1).Name + ':'
Set-Location -Path $sccmSite
```

The script imports the SCCM Powershell module which is installed as part of the SCCM Admin Console.  Unfortunately it's not available by powershell autoloader by default.  Once the module is loaded, the script working directory is set to the defalt SCCM site name via `Set-Location`

``` powershell
$puppetenc = (@"
{
  "config_mgr": {
    "environments_folder": "Puppet ENC\\Puppet Environments",
    "roles_folder": "Puppet ENC\\Puppet Roles",
    "profiles_folder": "Puppet ENC\\Puppet Profiles",
    "classes_folder": "Puppet ENC\\Puppet Classes",
    "root_limiting_collection": "All Systems",

    "environments_collection_prefix": "Puppet::Environment::",
    "roles_collection_prefix": "Puppet::Role::",
    "profiles_collection_prefix": "Puppet::Profile::",
    "Classes_collection_prefix": "Puppet::Class::"
  },
  
  "environments": [ "production","test" ],
  
  "roles": [
    {
      "name": "AdvWorks-Load-Balancer",
      "profiles" : [ "HAProxyService" ]
    },
    {
      "name": "AdvWorks-WebServer",
      "profiles" : [ "IISWebServer","IISBaselineSecurity","AdvWorksWebsite" ]
    },
    {
      "name": "AdvWorks-Database",
      "profiles" : [ "MSSQLServer","MSSQLServerBaselineSecurity","AdvWorksDatabase" ]
    }
  ],
  
  "profiles": [
    {
      "name": "HAProxyService",
      "classes": [ "haproxy" ]
    },
    {
      "name": "IISWebServer",
      "classes": [ "shared::iis","shared::iis::no_default_website" ]
    },
    {
      "name": "IISBaselineSecurity",
      "classes": [ "shared::iis::security" ]
    },
    {
      "name": "AdvWorksWebsite",
      "classes": [ "advworks::website" ]
    },
    {
      "name": "MSSQLServer",
      "classes": [ "shared::mssql" ]
    },
    {
      "name": "MSSQLServerBaselineSecurity",
      "classes": [ "shared::mssql::security" ]
    },
    {
      "name": "AdvWorksDatabase",
      "classes": [ "advworks::database" ]
    }
  ]
}
"@ | ConvertFrom-JSON -ErrorAction Stop)

if ($puppetenc -eq $null) { Throw "Invalid JSON" }
```

The environment-role-profile-class hierarchy is specified in a single text blob, represented via JSON and then converted into a Powershell Custom Object.  This makes it easy to read and modify the hierarchy as need.  You can use different techniques (hashtables, CSV files) but this is a method that I prefer.

The SCCM settings are defined at the top the JSON document; e.g. Limiting Collection, folder location for different collection types

``` powershell
# Helper function for creating a collection refresh schedule
Function New-RandomSchedule()
{
  "01/01/2000 $((Get-Random -Min 0 -Max 23).ToString('00')):$((Get-Random -Min 0 -Max 59).ToString('00')):00"
}
```

The collections are created with a daily refresh schedule.  This helper function just generates a random time of the day and is used later on during collection creation.

``` powershell
Write-Host "Processing Collections..."

# Create the environments
$puppetenc.environments | % {
  $EnvName = $_
  $CollectionName = "$($puppetenc.config_mgr.environments_collection_prefix)$EnvName"
  
  $thisColl = Get-CMDeviceCollection -Name $CollectionName
  if ($thisColl -eq $null) {
    Write-Host "Creating collection $CollectionName ..."

    $Schedule = New-CMSchedule -Start (New-RandomSchedule) -RecurInterval Days -RecurCount 1    
    $thisColl = New-CMDeviceCollection -Name $CollectionName -LimitingCollectionName $puppetenc.config_mgr.root_limiting_collection -RefreshType Periodic -RefreshSchedule $Schedule
    
    Move-CMObject -InputObject $thisColl -FolderPath "$sccmSite\DeviceCollection\$($puppetenc.config_mgr.environments_folder)" | Out-Null    
  }
}

# Create the roles
$puppetenc.roles | % {
  $Role = $_.name
  $CollectionName = "$($puppetenc.config_mgr.roles_collection_prefix)$Role"
  
  $thisColl = Get-CMDeviceCollection -Name $CollectionName
  if ($thisColl -eq $null) {
    Write-Host "Creating collection $CollectionName ..."

    $Schedule = New-CMSchedule -Start (New-RandomSchedule) -RecurInterval Days -RecurCount 1    
    $thisColl = New-CMDeviceCollection -Name $CollectionName -LimitingCollectionName $puppetenc.config_mgr.root_limiting_collection -RefreshType Periodic -RefreshSchedule $Schedule
    
    Move-CMObject -InputObject $thisColl -FolderPath "$sccmSite\DeviceCollection\$($puppetenc.config_mgr.roles_folder)" | Out-Null    
  }
}

# Create the profiles
$puppetenc.profiles | % {
  $PupProfile = $_.name
  $CollectionName = "$($puppetenc.config_mgr.profiles_collection_prefix)$PupProfile"
  
  $thisColl = Get-CMDeviceCollection -Name $CollectionName
  if ($thisColl -eq $null) {
    Write-Host "Creating collection $CollectionName ..."

    $Schedule = New-CMSchedule -Start (New-RandomSchedule) -RecurInterval Days -RecurCount 1    
    $thisColl = New-CMDeviceCollection -Name $CollectionName -LimitingCollectionName $puppetenc.config_mgr.root_limiting_collection -RefreshType Periodic -RefreshSchedule $Schedule
    
    Move-CMObject -InputObject $thisColl -FolderPath "$sccmSite\DeviceCollection\$($puppetenc.config_mgr.profiles_folder)" | Out-Null    
  }
}

# Generate the classes list
$puppetenc.profiles | % { $_.classes | % { Write-Output $_ } } | Select -Unique | % {
  $ClassName = $_
  $CollectionName = "$($puppetenc.config_mgr.classes_collection_prefix)$ClassName"
  
  $thisColl = Get-CMDeviceCollection -Name $CollectionName
  if ($thisColl -eq $null) {
    Write-Host "Creating collection $CollectionName ..."

    $Schedule = New-CMSchedule -Start (New-RandomSchedule) -RecurInterval Days -RecurCount 1    
    $thisColl = New-CMDeviceCollection -Name $CollectionName -LimitingCollectionName $puppetenc.config_mgr.root_limiting_collection -RefreshType Periodic -RefreshSchedule $Schedule
    
    Move-CMObject -InputObject $thisColl -FolderPath "$sccmSite\DeviceCollection\$($puppetenc.config_mgr.classes_folder)" | Out-Null    
  }
}
```

The script will now create all of the required collections e.g.

- For each role in the JSON file...
  -  Append the role prefix text to the name of the role
     e.g. The role called `AdvWorks-WebServer` will have a collection named `Puppet::Role::AdvWorks-WebServer`
  -  Check if the collection already exists
     If not, create the collection and move it to the roles collection folder.
     In this example the folder is called `Puppet ENC\Puppet Roles`

The Class collections are a little different.  The class names are extracted via the the Profiles;

> For each Profile, extract the class names and generate a unique list of classes.

Once all of the collections are created, the script then updates the collection memberships

``` powershell
Write-Host "Processing Collection Memberships..."
# Associate profiles to roles
$puppetenc.roles | % {
  $Role = $_.name
  $RoleCollectionName = "$($puppetenc.config_mgr.roles_collection_prefix)$Role"
  
  $roleColl = Get-CMDeviceCollection -Name $CollectionName
  if ($roleColl -eq $null) { throw "Missing Role"}

  $_.profiles | % { 
    $PupProfile = $_
    $ProfileCollectionName = "$($puppetenc.config_mgr.profiles_collection_prefix)$PupProfile"

    $includeRule = Get-CMDeviceCollectionIncludeMembershipRule -CollectionName $ProfileCollectionName -IncludeCollectionName $RoleCollectionName

    if ($includeRule -eq $null) {
      Write-Host "Adding $Role to $PupProfile"
      Add-CMDeviceCollectionIncludeMembershipRule -CollectionName $ProfileCollectionName -IncludeCollectionName $RoleCollectionName | Out-Null
    }
  }
}

$puppetenc.profiles | % {
  $PupProfile = $_.name
  $ProfileCollectionName = "$($puppetenc.config_mgr.profiles_collection_prefix)$PupProfile"
  
  $ProfileColl = Get-CMDeviceCollection -Name $CollectionName
  if ($ProfileColl -eq $null) { throw "Missing Profile"}

  $_.classes | % { 
    $ClassName = $_
    $ClassCollectionName = "$($puppetenc.config_mgr.classes_collection_prefix)$ClassName"

    $includeRule = Get-CMDeviceCollectionIncludeMembershipRule -CollectionName $ClassCollectionName -IncludeCollectionName $ProfileCollectionName

    if ($includeRule -eq $null) {
      Write-Host "Adding $PupProfile to $ClassName"
      Add-CMDeviceCollectionIncludeMembershipRule -CollectionName $ClassCollectionName -IncludeCollectionName $ProfileCollectionName | Out-Null
    }
  }
}
```

The script then uses the puppet JSON file to create the required collection membership rules e.g.  The `IISWebServer` Profile collection has an include rule for the `AdvWorks-WebServer` collection

### What does SCCM look like now?

Script output;

```
PS C:\> .\CreateSCCMInfo.ps1
Processing Collections...
Creating collection Puppet::Environment::production ...
Creating collection Puppet::Environment::test ...
Creating collection Puppet::Role::AdvWorks-Load-Balancer ...
Creating collection Puppet::Role::AdvWorks-WebServer ...
Creating collection Puppet::Role::AdvWorks-Database ...
Creating collection Puppet::Profile::HAProxyService ...
Creating collection Puppet::Profile::IISWebServer ...
Creating collection Puppet::Profile::IISBaselineSecurity ...
Creating collection Puppet::Profile::AdvWorksWebsite ...
Creating collection Puppet::Profile::MSSQLServer ...
Creating collection Puppet::Profile::MSSQLServerBaselineSecurity ...
Creating collection Puppet::Profile::AdvWorksDatabase ...
Creating collection Puppet::Class::haproxy ...
Creating collection Puppet::Class::shared::iis ...
Creating collection Puppet::Class::shared::iis::no_default_website ...
Creating collection Puppet::Class::shared::iis::security ...
Creating collection Puppet::Class::advworks::website ...
Creating collection Puppet::Class::shared::mssql ...
Creating collection Puppet::Class::shared::mssql::security ...
Creating collection Puppet::Class::advworks::database ...
Processing Collection Memberships...
Adding AdvWorks-Load-Balancer to HAProxyService
Adding AdvWorks-WebServer to IISWebServer
Adding AdvWorks-WebServer to IISBaselineSecurity
Adding AdvWorks-WebServer to AdvWorksWebsite
Adding AdvWorks-Database to MSSQLServer
Adding AdvWorks-Database to MSSQLServerBaselineSecurity
Adding AdvWorks-Database to AdvWorksDatabase
Adding HAProxyService to haproxy
Adding IISWebServer to shared::iis
Adding IISWebServer to shared::iis::no_default_website
Adding IISBaselineSecurity to shared::iis::security
Adding AdvWorksWebsite to advworks::website
Adding MSSQLServer to shared::mssql
Adding MSSQLServerBaselineSecurity to shared::mssql::security
Adding AdvWorksDatabase to advworks::database
PS CEN:\>
```

We then add a few systems to the Roles;

- 1 x Linux system to the `AdvWorks-Load-Balancer` role collection

- 2 x Windows systems to the `AdvWorks-WebServer` role collection

- 1 x Windows system to the `AdvWorks-Database` role collection

- All four systems are added to the `production` environment collection.  Once the collection memberships are refreshed, SCCM should look like;

*Environment Collections*
![Environment Collections]({{ site.url }}/blog-images/sccm-puppet-enc-003.png)

*Role Collections*
![Role Collections]({{ site.url }}/blog-images/sccm-puppet-enc-004.png)

*Profile Collections*
![Role Collections]({{ site.url }}/blog-images/sccm-puppet-enc-005.png)

*Class Collections*
![Role Collections]({{ site.url }}/blog-images/sccm-puppet-enc-006.png)


## What's next?

In the next post we'll create a simple web service to query SCCM Database, and generate the required YAML for a Puppet ENC.
