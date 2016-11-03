---
title: Using Powershell to query Neo4j using Bolt
excerpt: Using Powershell to query Neo4j using Bolt and native .Net Framework Graph Database driver
category:
  - blog
header:
  image: header-neo4j-globe.png
  overlay_image: header-neo4j-globe.png
  overlay_filter: 0.5
  teaser: teaser-ps-and-neo.png
tags:
  - neo4j
  - powershell
  - dotnet
  - bolt
---

[Previously]({{ site.url }}/blog/powershell-neo4j-metrics), we used the HTTP REST endpoint to query Neo4j however with v3.0, Neo4j also offers a binary level transport called [Bolt](https://neo4j.com/docs/developer-manual/current/terminology/#term-bolt).  This has an added benefit of proper types instead of the limited set offered in JSON.

Neo4j also support native drivers which includes a .Net driver, which can be be used in Powershell.  But setting this up can a little tricky.

## Download the Neo4j.Driver nuget package

The .Net driver is available as Nuget package on the [Nuget Gallery](https://www.nuget.org/packages/Neo4j.Driver).  To install the driver to the current directory run the following command:

``` batch
nuget install Neo4j.Driver
```

This will install the latests stable version of the driver.  At the time of this blog post, that is 1.0.2

If you do not have nuget installed it can be downloaded from [https://dist.nuget.org/index.html](https://dist.nuget.org/index.html)


## Use Powershell 

We can then use a simple Powershell script to load the Neo4j.Driver library, connect to Neo4j and then run a simple query

``` powershell
# Import DLLs
Add-Type -Path "$PSScriptRoot\Neo4j.Driver.1.0.2\lib\dotnet\Neo4j.Driver.dll"
Add-Type -Path "$PSScriptRoot\rda.SocketsForPCL.1.2.2\lib\net45\Sockets.Plugin.Abstractions.dll"
Add-Type -Path "$PSScriptRoot\rda.SocketsForPCL.1.2.2\lib\net45\Sockets.Plugin.dll"

$authToken = [Neo4j.Driver.V1.AuthTokens]::Basic('neo4j','password')

$dbDriver = [Neo4j.Driver.V1.GraphDatabase]::Driver("bolt://localhost:7687",$authToken)
$session = $dbDriver.Session()
try {
  $result = $session.Run("MATCH (n) RETURN Count(n) AS NumNodes")

  Write-Host ($result | ConvertTo-JSON -Depth 5)
} finally {
  $session = $null
  $dbDriver = $null
}
```

I have loaded the sample Movies database into Neo4j so the script gives the following output:

```
{                                           
    "Values":  {                            
                   "NumNodes":  171         
               },                           
    "Keys":  [                              
                 "NumNodes"                 
             ]                              
}
```

## Use Powershell - in more detail...

So let's go through the script in more detail

---

``` powershell
# Import DLLs
Add-Type -Path "$PSScriptRoot\Neo4j.Driver.1.0.2\lib\dotnet\Neo4j.Driver.dll"
Add-Type -Path "$PSScriptRoot\rda.SocketsForPCL.1.2.2\lib\net45\Sockets.Plugin.Abstractions.dll"
Add-Type -Path "$PSScriptRoot\rda.SocketsForPCL.1.2.2\lib\net45\Sockets.Plugin.dll"
```

Powershell allows you to load .Net Framework assemblies using the `Add-Type` cmdlet.  The Neo4j.Driver has a different number of folders in the `lib` directory, each for a different platform or .Net Framework distribution.  Neo4j.Driver depends on the `rda.Sockets` package which is automatically installed by nuget.  It too has many folders for different platforms etc.  Typically Visual Studio would use the correct libraries when compiling a project but in our case we need to do this ourselves.

In my case I was using Windows 10 with the full .Net Framework 4.6 so I could use the `dotnet` library from the `Neo4j.Driver` package and the `net45` library from the `rda.Sockets` package.  Once these DLLs are loaded they can be used in the Powershell session.

---

``` powershell
$authToken = [Neo4j.Driver.V1.AuthTokens]::Basic('neo4j','password')
```

The Neo4j database has authentication enabled so we need to create an authentication token (`AuthToken`) object.  Currently [only a Basic token](https://github.com/neo4j/neo4j-dotnet-driver/blob/1.0/Neo4j.Driver/Neo4j.Driver/V1/IAuthToken.cs#L53-L70) is supported but more may be available in the future.  We simply create a new object (Could also use the `New-Object` cmdlet too) with a username and password

---

``` powershell
$dbDriver = [Neo4j.Driver.V1.GraphDatabase]::Driver("bolt://localhost:7687",$authToken)
$session = $dbDriver.Session()
```

Once we have an authentication token, we can connect to the database using the `Driver` object.  The connection uri follows the typical URI format of `<protocol>://<hostname>:<port>`, so in the script above; Connect over the bolt protocol to `localhost` over port 7687 (default port for bolt)

Once connect we can retrive a [Session](https://github.com/neo4j/neo4j-dotnet-driver/blob/1.0/Neo4j.Driver/Neo4j.Driver/V1/IDriver.cs#L37-L45) object so we can run queries.

---

``` powershell
try {
  $result = $session.Run("MATCH (n) RETURN Count(n) AS NumNodes")

  Write-Host ($result | ConvertTo-JSON -Depth 5)
} finally {
  $session = $null
  $dbDriver = $null
}
```

With the `Session` object we can run arbitrary cypher commands, including stored procedures ([APOC](https://neo4j.com/blog/intro-user-defined-procedures-apoc/)).  The return object from the cypher command is of type `IEnumerable` which powershell understands natively so we can use the `Convert-JSON` command to serialise it into JSON.  But we could just as easily use `For-EachObject` too, for example;

``` powershell
  $result = $session.Run("MATCH (n)-[]->(m) RETURN n,m LIMIT 10")

  $result | ForEach-Object {
    Write-Host ("From " + ($_['n']).Labels + " To " + ($_['m']).Labels)
  }
```

This query retrieves 10 matches for one node to another (from `n` to `m`).  We can then use the `For-EachObject` cmdlet to parse over the result set (10 returned rows) and then extract the labels for the return node called n (`$_['n']).Labels`) and m (`$_['m']).Labels`.


## Conclusion

Once the .Net driver DLLs are loaded into the Powershell session it's fairly easy to query and then parse the results of Cypher queries using the new, and much faster bolt protocol.

In the next blog post, I will build upon [the Powershell script to query the Neo4j metrics]({{ site.url }}/blog/powershell-neo4j-metrics), and use Bolt with inbuilt stored procedures instead.
