---
title: Graph all the PowerShell things!
excerpt: Using PowerShell with Neo4j - A java based graph database
category:
  - blog
header:
  overlay_image: /images/header-neo4j-globe.png
  overlay_filter: 0.75
  teaser: /images/teaser-ps-and-neo.png
tags:
  - neo4j
  - powershell
---

{% include custom/mermaid.html %}

At the PowerShell and DevOps Global Summit 2017 I had two community lightning talks available.  Due to how many awesome talks there were, there wasn't enough time to do my talk on Neo4j and PowerShell.  So instead, I decided to write this blog post.

---

## Graph all the PowerShell things!

I am currently a Senior Software Developer on the Modules with Puppet, specialising in Windows. I am also a contributor to Neo4j which is a Graph Database.  [Neo4j](https://neo4j.com/) is a highly scalable, native graph database purpose-built to leverage not only data but also its relationships, but this seems a very odd place to find PowerShell.

Strangely enough, there is a lot you can do with PowerShell and Neo4j.

### Chocolatey and PSake

I maintain the [Chocolatey](https://chocolatey.org/) packages for the community edition of Neo4j, and of course being Chocolatey it's all PowerShell.  However I'm also lazy.  Neo4j were doing a lot of releases and I had trouble keeping up, so PowerShell to the rescue.  I created a PSake build task which would "screen scrape" the Neo4j Downloads webpage and gather which versions were the latest and their relevant download links.  The task would then see if I had not packaged this version yet.  If not, it would create the packaging template, commit the changes the github repository, create the chocolatey package and then publish it.

So now I had it automated, but I still had to remember to actually run the script and I needed a computer to run the script on.  Appveyor to the rescue!!  Appveyor is cloud based Continuous Integration/Build Service using Windows worker agents.  It's also free to use with Open Source projects which was perfect.  I created an AppVeyor configuration which runs my PSake task once a day.  So now I have Chocolatey packages, automatically created and published and I don't need to lift a finger!  I also get alerts if the script fails during the process, so I know when I need to update my templates for a new Neo4j release.

#### Links

* [Neo4j packages on Chocolatey](https://chocolatey.org/packages/neo4j-community/3.1.4)

* [Source for packages](https://github.com/glennsarti/neo4j-community-chocolatey)

* [PSake build file](https://github.com/glennsarti/neo4j-community-chocolatey/blob/master/default.ps1)


### PowerShell Module and Pester

Previously Neo4j used some simple batch files to manage Neo4j on Windows, hoever they were far from optimal.  Neo4j asked me to help them out and rewrite the management scripts in PowerShell.  This meant that we could have far more intelligent scripts but also test them using Pester, a unit testing framework for PowerShell.

So when you use Neo4j on Windows, you're actually using a PowerShell module, called `Neo4j-Management`.  In fact, if you want to, you can import this module directly instead of using the wrapper batch files!

#### Links

* [Neo4j-Management private PowerShell Module](https://github.com/neo4j/neo4j/tree/3.2/packaging/standalone/src/main/distribution/shell-scripts/bin/Neo4j-Management)

* [Pester tests for module](https://github.com/neo4j/neo4j/tree/3.2/packaging/standalone/src/tests/Neo4j-Management)


### Graphing PowerShell Help

The PowerShell help system is one of the jewels of the system, and like most documentation systems has many relationships between help documents.  This is perfect for a graph database, in fact Neo4j was first developed by Emil Eifren as a solution to enterprise documentation management.

So as an example in how to use Neo4j with PowerShell, I set out to graph the help documents in PowerShell and the relationships between them.  PowerShell help items have the ability to link to other items, in particular the `RELATED LINKS` section, for example:

```powershell
C:\> get-help get-item

NAME
    Get-Item
...
RELATED LINKS
    Online Version: http://go.microsof ...
    Clear-Item
    Copy-Item
    Invoke-Item
    Move-Item
...
```

Using this information we can construct a simple graph schema:

<div class="mermaid">
graph LR;
  module1(Module)-- HAS_COMMAND -->cmdlet1(Command)
  cmdlet2(Command)-- RELATED_LINK -->cmdlet3(Command)

  style module1 fill:#E0E0F0,stroke:#0000FF;
  style cmdlet1 fill:#E0F0E0,stroke:#00FF00;
  style cmdlet2 fill:#E0F0E0,stroke:#00FF00;
  style cmdlet3 fill:#E0F0E0,stroke:#00FF00;
</div>

* A Module has one or more Commands (also known as cmdlet or function).  This is the `HAS_COMMAND` relationship.

* A Command can be related to zero or more other Commands.  This is the `RELATED_LINK` relationship.

#### Getting ready to create the graph

##### Installing Neo4j

Use the Neo4j Chocolatey package:

```powershell
PS> choco install neo4j-community
```

This will install the a Neo4j Graph Database.  When you first browse to `http://localhost:7474` you will be asked to setup the default Neo4j user.  By default this is username and password of `neo4j`.  The code for this blog post assumes you change the the password of the neo4j user to `Password1`.  Of course, you can use your own username and password, just remember to change that in the scripts for this blog post.

##### Getting the nuget packages

* Get the source code from [code-glennsarti.github.io](https://github.com/glennsarti/code-glennsarti.github.io)
* Change the directory to `graphing-powershell-help`
* Create a new directory called `nuget` and change directory into that
* Install the Neo4j.Driver nuget package.  This blog post specifically uses version 1.0.2 as version 2.0+ has different dependencies which are more difficult to resolve in PowerShell.

```powershell
PS> git clone https://github.com/glennsarti/code-glennsarti.github.io.git
...
PS> cd code-glennsarti.github.io
PS> New-Item nuget -ItemType Directory
PS> cd nuget
PS> nuget install Neo4j.Driver -version 1.0.2
```

Note - If you do not have nuget installed, download it from [nuget.org](https://dist.nuget.org/index.html).

#### Creating the graph

The `Import-PowerShell-Help.ps1` script imports all of the help information into the Neo4j graph database.  This is a two pass import script, which could be improved to be a lot faster but would make it harder to explain.  The first pass imports all of the modules and commands, and the second pass adds the relationships between commands.

Source Code - [Import-PowerShell-Help.ps1](https://github.com/glennsarti/code-glennsarti.github.io/blob/master/graphing-powershell-help/Import-PowerShell-Help.ps1)

---
Calls `Update-Help` so that all of the latest help is available on the local computer

```powershell
$ErrorActionPreference = 'Stop'

# Update all of the help files
Update-Help -Confirm:$false
```

Imports all of the modules into the PowerShell session.

This can take a while and may consume a lot of memory
{: .notice--warning}

```powershell
# Import the modules into the session
Get-Module -List | Select-Object -Unique | % { 
  Write-Host "Importing $($_.Name)"
  $_ | Import-Module -ErrorAction Continue
}
```

Imports the Neo4j.Driver DLLs and connects to the Neo4j database

```powershell
# Import DLLs
Add-Type -Path "$PSScriptRoot\nuget\Neo4j.Driver.1.0.2\lib\dotnet\Neo4j.Driver.dll"
Add-Type -Path "$PSScriptRoot\nuget\rda.SocketsForPCL.1.2.2\lib\net45\Sockets.Plugin.Abstractions.dll"
Add-Type -Path "$PSScriptRoot\nuget\rda.SocketsForPCL.1.2.2\lib\net45\Sockets.Plugin.dll"

Function Invoke-Cypher($query) {
  $session.Run($query)
}

$authToken = [Neo4j.Driver.V1.AuthTokens]::Basic('neo4j','Password1')

$dbDriver = [Neo4j.Driver.V1.GraphDatabase]::Driver("bolt://localhost:7687",$authToken)
$session = $dbDriver.Session()
```

You can optionally only query certain modules or commands.  By default it consumes everything

```powershell
$moduleFilter = '.+'
$commandFilter = '.+'
```

Deletes all previous nodes and relationships

```powershell
try {
  # Kill everything ...
  $result = Invoke-Cypher("MATCH ()-[r]-() DELETE r")
  $result = Invoke-Cypher("MATCH (n) DELETE n")
```

Enumerates all of the commands for each module and creates the `Module` and `Command` nodes in the graph database.

```powershell
  # Create all the cmdlets and modules
  Get-Module | ? { $_.Name -match $moduleFilter } | ForEach-Object -Process {
    $ModuleName = $_.name
    Write-Progress -Activity "Parsing $ModuleName" -Status "Importing Commands"
    Invoke-Cypher("CREATE (:Module { name:'$ModuleName'})")

    Get-Command -Module $_ | ? { $_.Name -match $commandFilter } | ForEach-Object -Process {
      $CommandName = $_.Name
      $query = "MATCH (m:Module { name:'$ModuleName'})`n" + `
               "CREATE (com:Command { name:'$CommandName'})`n" + `
               "  SET com.commandtype = '$($_.CommandType)'`n" + `
               "WITH m,com`n" + `
               "CREATE (m)-[:HAS_COMMAND]->(com)"
      Invoke-Cypher($query)
    }
  }
```

Enumerates the help for each command, ignoring anything that has a URL associated (Online Help).  The script the creates a `RELATED_LINK` relationship between the nodes.

```powershell
  # Create all the cmdlets and modules
  Get-Module | ? { $_.Name -match $moduleFilter } | ForEach-Object -Process {
    $ModuleName = $_.name
    Write-Progress -Activity "Parsing $ModuleName"
    Get-Command -Module $_ | ? { $_.Name -match $commandFilter } | ForEach-Object -Process {
      $ThisCommandName = $_.Name

      Write-Progress -Activity "Parsing $ModuleName" -Status "Creating links for $ThisCommandName"
      $thisURI = $null
      (Get-Help $_).relatedLinks.navigationLink | ForEach-Object -Process {
        $HelpLink = $_

        # Ignore anything that is a real URI
        if ($HelpLink.uri -eq '') {
          $ThatCommandName = $HelpLink.linkText

          $query = "MATCH (this:Command { name:'$ThisCommandName'})`n" + `
                   "MATCH (that:Command { name:'$ThatCommandName'})`n" + `
                   "CREATE (this)-[:RELATED_LINK]->(that)"
          Invoke-Cypher($query)
        } else {
          $thisURI = $HelpLink.uri
        }
      }

      if ($thisURI -ne $null) {
        $query = "MATCH (this:Command { name:'$ThisCommandName'})`n" + `
                 "SET this.uri = '$thisURI'`n"
        Invoke-Cypher($query)
      }
    }
  }
} finally {
  $session = $null
  $dbDriver = $null
}
```

This can take a while and may consume a lot of memory
{: .notice--warning}


#### Querying the graph

The `Example-Queries.ps1` script then queries the graph database for interesting information.  The `Invoke-Cypher` function while strictly not needed, makes outputing the results of [cypher](https://neo4j.com/developer/cypher-query-language/) queries eaier and look prettier.

Source Code - [Example-Queries.ps1](https://github.com/glennsarti/code-glennsarti.github.io/blob/master/graphing-powershell-help/Example-Queries.ps1)

Here's what the output from my local computer looks like;

---

There were 79 modules and 1656 commands imported

```powershell
C:\Source\code-glennsarti.github.io\graphing-powershell-help [add-neo4j-ps +1 ~0 -0 !]> .\Example-Queries.ps1

List the entities
-----------------
MATCH (m:Module)
WITH COUNT(m) AS ModuleCount
MATCH (c:Command)
WITH ModuleCount,COUNT(c) AS CommandCount
RETURN ModuleCount,CommandCount, SIZE(()-[:RELATED_LINK]->()) AS LinkCount

ModuleCount LinkCount CommandCount
----------- --------- ------------
         79      5181         1656
```

The Hyper-V modules had the most number of commands.

```powershell
List all modules
-----------------
MATCH (m:Module)-[:HAS_COMMAND]->(com:Command)
RETURN m.name AS ModuleName, Count(com) AS Commands
ORDER BY Count(com) DESC
LIMIT 10

ModuleName                      Commands
----------                      --------
Hyper-V                              232
Storage                              154
Microsoft.PowerShell.Utility         107
Microsoft.PowerShell.Management       89
netsecurity                           85
NetAdapter                            68
Dism                                  43
MsDtc                                 41
NetEventPacketCapture                 35
SmbShare                              35
```

There were many commands that linked to themselves.  This seems like a bad user experience e.g. For more help about Get-Process, see Get-Process.

```powershell
Commands that link themselves
-----------------
MATCH (this:Command)-[:RELATED_LINK]->(that:Command)
WHERE this = that
RETURN this.name
LIMIT 10

this.name
---------
Get-DnsClientNrptPolicy
Debug-Process
Get-Process
Limit-EventLog
New-EventLog
Remove-Computer
Remove-EventLog
Stop-Process
Wait-Process
Write-EventLog
```

Many commands had no inbound or outbound links which again, may be a bad user experience.

```powershell
Commands that have no links at all (Hermits)
-----------------
MATCH (this:Command)
WHERE NOT ( (this)-[:RELATED_LINK]-(:Command) )
RETURN this.name
LIMIT 10

this.name
---------
Use-WindowsUnattend
Resolve-DnsName
New-EtwTraceSession
Grant-HgsKeyProtectorAccess
Revoke-HgsKeyProtectorAccess
Get-HgsAttestationBaselinePolicy
Add-VMAssignableDevice
Add-VMDvdDrive
Add-VMFibreChannelHba
Add-VMGpuPartitionAdapter
```

Surprisingly the Save-NetGPO and Open-NetGPO were the most popular commands, whereby they had vastly more commands linking to them, than they did linking to others.

```powershell
Popular Commands.  More inbound links than outbound
-----------------
MATCH (this:Command)
WHERE SIZE( (this)<-[:RELATED_LINK]-(:Command) ) >
      SIZE( (this)-[:RELATED_LINK]->(:Command) )
RETURN
  this.name,
  SIZE( (this)<-[:RELATED_LINK]-(:Command) ) AS InboundLinks,
  SIZE( (this)-[:RELATED_LINK]->(:Command) ) AS OutboundLinks,
  (SIZE( (this)<-[:RELATED_LINK]-(:Command) ) - SIZE( (this)-[:RELATED_LINK]->(:Command) )) AS LinksDiff
ORDER BY LinksDiff DESC
LIMIT 10

LinksDiff OutboundLinks this.name                    InboundLinks
--------- ------------- ---------                    ------------
       57             3 Save-NetGPO                            60
       55             5 Open-NetGPO                            60
       15             4 Get-NetFirewallProfile                 19
       14             0 Get-VM                                 14
       14            12 Get-NetFirewallAddressFilter           26
       13             6 Get-NetFirewallPortFilter              19
       12             1 Disable-BC                             13
       12             1 Reset-BC                               13
       12             7 Get-Disk                               19
       12             7 Set-NetIPsecRule                       19
```

Also surprising was that were many cases of commands linking to commands in a different module.

```powershell
Links across Modules
-----------------
MATCH (thismodule:Module)-[:HAS_COMMAND]->(this:Command)-[r:RELATED_LINK]->(that:Command)<-[:HAS_COMMAND]-(thatmodule:Module)
WHERE (this <> that) AND (thismodule <> thatmodule)
RETURN
  ("(" + thismodule.name + ") " + this.name) AS From,
  ("(" + thatmodule.name + ") " + that.name) AS To
LIMIT 10

From                                       To
----                                       --
(Microsoft.PowerShell.Utility) Get-Culture (International) Set-Culture
(PKI) Switch-Certificate                   (Microsoft.PowerShell.Management) Set-Location
(PKI) Import-PfxCertificate                (Microsoft.PowerShell.Management) Set-Location
(PKI) Import-Certificate                   (Microsoft.PowerShell.Management) Set-Location
(PKI) Get-Certificate                      (Microsoft.PowerShell.Management) Set-Location
(PKI) Test-Certificate                     (Microsoft.PowerShell.Management) Get-ChildItem
(PKI) Switch-Certificate                   (Microsoft.PowerShell.Management) Get-ChildItem
(PKI) Import-PfxCertificate                (Microsoft.PowerShell.Management) Get-ChildItem
(PKI) Import-Certificate                   (Microsoft.PowerShell.Management) Get-ChildItem
(PKI) Get-Certificate                      (Microsoft.PowerShell.Management) Get-ChildItem
```

To show the recommendation engine side the Graph Database, inside of the database I created a module called `Glenn` with a single command called `Invoke-Glenn` which had a link to the `New-JobTrigger` command.  I then used a simple recommendation query to suggest other commands I should link too as well.

```powershell
MATCH (that:Command {name: 'New-JobTrigger'})
MERGE (mod:Module { name: 'Glenn'})-[:HAS_COMMAND]->(glenn:Command {name: 'Invoke-Glenn'})
MERGE (glenn)-[:RELATED_LINK]->(that)

Recommendation Engine (New-JobTrigger)
-----------------
MATCH
  (this:Command)-[:RELATED_LINK*2..4]->(recommend:Command)
WHERE
  (this.name = 'Invoke-Glenn')
  AND (this <> recommend)
  AND NOT ( (this)-[:RELATED_LINK]->(recommend) )
WITH
  recommend, COUNT(recommend) AS Popularity
RETURN
  recommend.name AS CommandName,Popularity
ORDER BY Popularity DESC
LIMIT 5

Popularity CommandName
---------- -----------
       270 Set-ScheduledJobOption
       270 Get-JobTrigger
       270 Disable-ScheduledJob
       270 Remove-JobTrigger
       270 Register-ScheduledJob
```

### Closing thoughts

The point of this blog/presentation was not really about how Neo4j and PowerShell play together.  It was more that people can use their PowerShell skills for more than just system administration.  There are many open source community projects that lack Windows skills which we in the PowerShell community can help with, and hopefully bring awesome products, like Neo4j into the Windows fold.
