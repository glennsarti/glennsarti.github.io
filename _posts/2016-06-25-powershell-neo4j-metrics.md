---
title: Querying Neo4j with Powershell - Metrics
excerpt: Powershell function to query Neo4j metrics via JMX
category:
  - blog
tags:
  - neo4j
  - powershell
---

[Neo4j](https://neo4j.com/) is a highly scalable, native graph database purpose-built to leverage not only data but also its relationships.  While running the database is fairly simple, like all systems it's important to monitor the database.

## Neo4j metrics

Neo4j has provided some metrics in the web based console.  For example, running `:play sysinfo` will display a basic set of metrics;

![neo4j-play-sysinfo]({{ site.url }}/blog-images/neoj-ps-001.png)

The console also has more advanced metrics available in the favourites side bar;

![image-left]({{ site.url }}/blog-images/neoj-ps-002.png){: .align-left}

Server Configuration

Kernel Information

ID Allocation

Store file sizes

Extensions


### Metrics REST API

Running the `Kernel Information` query will display more advanced metrics about the Neo4j, for example the version of Neo4j;

``` json
...
      {
        "isIs": "false ",
        "name": "KernelVersion",
        "description": "The version of Neo4j",
        "type": "java.lang.String",
        "value": "neo4j-kernel, version: 3.0.1,0044e52",      <---- The Neo4j database version
        "isReadable": "true",
        "isWriteable": "false "
      },
...
```

But what was actually in that Neo4j query?

```
// Kernel information
:GET /db/manage/server/jmx/domain/org.neo4j/instance%3Dkernel%230%2Cname%3DKernel
```

This is not a usual [cypher](https://neo4j.com/developer/cypher-query-language/) query.  Instead it's a query against the JMX API provided by Neo4j.  [Java Management Extensions](https://en.wikipedia.org/wiki/Java_Management_Extensions) (JMX) are a Java technology that supplies tools for managing and monitoring applications.  The full JMX API and detailed information about Neo4j metrics, is documented on the [Neo4j website](https://neo4j.com/docs/operations-manual/current/#monitoring).

Because these metrics are available by a HTTP API, this means we can use other tools to query for metrics, such as Powershell.

## Structure of a JMX endpoint

Before we start with the powershell function it's important to understand the structure of the JMX API;

* The root of the API consists of a collection of Domains.  A domain is typically named after a Java namespace e.g. the Neo4j namespace is `org.neo4j` and so is the JMX domain.  Other examples of domain names are `java.util.logging` and `com.sun.management`.

* A domain is composed of multiple Beans.  For our purposes, a bean is a logical grouping of metrics e.g. the `org.neo4j:instance=kernel#0,name=Kernel` holds metrics about the kernel in Neo4j

* A bean is composed of multiple metric keys and values.  This is where the metrics are stored e.g. the `KernelVersion` metric holds the value the Neo4j kernel version 

# Querying with Powershell
I created a powershell function, `Get-Neo4jStats` which takes a few parameters and then queries the Neo4j JMX REST API.  It outputs a Powershell object with the metrics.

Source Code - [Get-Neo4jStats.ps1](https://github.com/glennsarti/code-glennsarti.github.io/blob/master/neo4j-powershell/Get-Neo4jStats.ps1)

## Input Parameters

### URL
The URL of the Neo4j REST API e.g. `http://127.0.0.1:7474`

### Credential
Neo4j requires authentication to interact with Neo4j.  By default this is `neo4j/neo4j`

### IncludeDomains
There are many domains exposed by JMX.  You can limit which domains to query e.g. `org.neo4j`


## Output Format
The output from the function is a structured Powershell Custom object;

```
+- domains 
+- <domain #1> +- beans
|              +- <bean #1> +- description
|              |            +- attributes
|              |
|              +- <bean #2> +- description
|              |            +- attributes
|              |
|              +- <bean xx>
|
+- <domain #2>
+- <domain xx>
```

`domains` An array of strings of the names of the domains in the output

`beans`  An array of strings of the names of the beans in the domain

`description` A text description of the bean

`attributes` A hashtable of names and values of the metrics for the bean

For example;

`'org.neo4j'.'org.neo4j:instance=kernel#0,name=Kernel'.attributes['KernelVersion']`

Would retrieve the value of a metric called `KernelVersion` in a bean called `org.neo4j:instance=kernel#0,name=Kernel` in a domain called `org.neo4j`


## How to use the powershell script

* Download the script from the [blog repository](https://raw.githubusercontent.com/glennsarti/code-glennsarti.github.io/master/neo4j-powershell/Get-Neo4jStats.ps1)

* Import the file into your powershell session e.g.

``` powershell
PS C:\> . C:\Get-Neo4jStats.ps1
```

* Create a Credential object

``` powershell
# Create it via a secure string
$secpasswd = ConvertTo-SecureString "neo4j" -AsPlainText -Force
$neo4jcreds = New-Object System.Management.Automation.PSCredential ('neo4j', $secpasswd)  

# ... or prompt the user
$neo4jcreds = Get-Credential -Message 'Neo4j Stats Credentials'
```

* Get help about the function

``` powershell
PS C:\> Get-Help Get-Neo4jStats
```


## Examples

### List all JMX domains

``` powershell
PS C:\> $stats = Get-Neo4jStats -Credential $neo4jcreds -URL 'http://127.0.0.1:7474'
PS C:\> $stats.domains
JMImplementation
java.util.logging
java.lang
com.sun.management
org.neo4j
java.nio
```


### List all JMX beans in the org.neo4j domain

``` powershell
PS C:\> $stats = Get-Neo4jStats -Credential $neo4jcreds -URL 'http://127.0.0.1:7474'
PS C:\> $stats.'org.neo4j'.beans
org.neo4j:instance=kernel#0,name=Kernel
org.neo4j:instance=kernel#0,name=Primitive count
org.neo4j:instance=kernel#0,name=Store file sizes
org.neo4j:instance=kernel#0,name=Configuration
```


### List all information about the Neo4j kernel

``` powershell
PS C:\> $stats = Get-Neo4jStats -Credential $neo4jcreds -URL 'http://127.0.0.1:7474'
PS C:\> $stats.'org.neo4j'.'org.neo4j:instance=kernel#0,name=Kernel'.attributes

Name                           Value
----                           -----
MBeanQuery                     org.neo4j:instance=kernel#0,name=*
KernelStartTime                Sat Jun 25 15:02:42 PDT 2016
StoreLogVersion                0
DatabaseName                   graph.db
ReadOnly                       False
StoreCreationDate              Mon May 30 17:22:31 PDT 2016
StoreId                        eeec5c5c57b48258
KernelVersion                  neo4j-kernel, version: 3.0.1,0044e52
```

### List all information about the Neo4j kernel

Remember back at the beginning of this post we used `:play sysinfo` to display some basic metrics.  We can use powershell to retrieve those similar values;

``` powershell
PS C:\> $stats = Get-Neo4jStats -Credential $neo4jcreds -URL 'http://127.0.0.1:7474'
PS C:\> $stats.'org.neo4j'.'org.neo4j:instance=kernel#0,name=Store file sizes'.attributes

Name                           Value
----                           -----
StringStoreSize                8192
PropertyStoreSize              2015273
RelationshipStoreSize          1117920
NodeStoreSize                  229320
LogicalLogSize                 10749055
TotalStoreSize                 15154000
ArrayStoreSize                 24576


PS C:\> $stats.'org.neo4j'.'org.neo4j:instance=kernel#0,name=Primitive count'.attributes

Name                           Value
----                           -----
NumberOfRelationshipIdsInUse   32467
NumberOfNodeIdsInUse           14758
NumberOfPropertyIdsInUse       48963
NumberOfRelationshipTypeIds... 8
```

Note that store sizes are in Bytes, so the `NodeStoreSize` of 229320 is 223.95KB


### Save the basic Neo4j metrics to disk as a JSON file

``` powershell
PS C:\> Get-Neo4jStats -Credential $neo4jcreds -URL 'http://127.0.0.1:7474' -IncludeDomains 'org.neo4j' | 
        ConvertTo-Json -Depth 10 |
        Out-File 'C:\neo4j.json'
```

# Script Limitations

The script has a fews issues with the JMX provider.  Fortunately most of these limitations are only in the standard Java JMX domains, not the Neo4j domain.

* Complex Java types can't be interpretted correctly

Unfortunately some attributes return complex Java types.  As the powershell script does not have enough information about the types it is unable to convert them to a powershell equivalent.

* Unable to accurately convert dates

The date strings returned do not include timezone information so it is not possible to convert the dates accurately.  Instead I just return the date types as a raw string.

* Unable to convert most array types

Much like the complex java types, it is too difficult to convert this to a Powershell equivalent. 
