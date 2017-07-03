---
title: Starting out with Puppet custom facts
excerpt: A quick start to converting external facts to ruby custom facts on Windows
category:
  - blog
header:
  image: /images/header-puppet.png
  overlay_image: /images/header-puppet.png
  overlay_filter: 0.5
  teaser: /images/teaser-puppet.png
tags:
  - ruby
  - windows
  - puppet
  - facter
  - powershell
---

{% include toc icon="tags" %}

## Puppet, PowerShell and Facter

Puppet uses a product called [Facter](https://docs.puppet.com/facter/latest/index.html) to gather system information during a puppet run.

> Facter is Puppet's cross-platform system profiling library. It discovers and reports per-node facts, which are available in your Puppet manifests as variables.

There are some [core facts](https://docs.puppet.com/facter/latest/core_facts.html) which are processed on all operating system but, two additional types of facts can be used to extend facter; External Facts and Custom Facts

**External facts**

[External](https://docs.puppet.com/facter/latest/custom_facts.html#external-facts) facts provide a way to use arbitrary executables or scripts as facts, or set facts statically with structured data. If you’ve ever wanted to write a custom fact in Perl, C, PowerShell, or a one-line text file, this is how.

**Custom Facts**

[Custom](https://docs.puppet.com/facter/latest/fact_overview.html) facts are written in ruby and have more advanced features and confinements.

Most people when starting out will write PowerShell (or heaven forbid, batch files) external facts which is a good start.  However there are some downsides.  Firstly the execution time PowerShell scripts can be a little slow.  This is mainly due to PowerShell itself starting.  The other downside is you can not tell Unix based operating systems to not try an execute these Windows specific external facts.  While there are some tricks to help stop this, it's far too easy for Windows scripts to log warnings and errors making it harder to figure out when real issues occur.

But the apparent learning curve to writing ruby looks steep; if all you want to do is read a registry key and output the result, why should a Windows administrator have to learn ruby?  Well, this blog post is to help reduce the effort it takes to write custom facts.  Also, if you squint, the ruby language looks a lot like (and in some cases operates similarly to) PowerShell.

The source code for these examples is available on my [blog github repo](https://github.com/glennsarti/code-glennsarti.github.io/tree/master/customfacts).

## Writing a registry based custom fact

### The external fact

For this example we'll convert a batch file based external fact to a ruby external fact.  This fact reads the `EditionID` of the operating system from the registry and then populates a fact called `Windows_Edition`.

```
@ECHO OFF
for /f "skip=1 tokens=3" %%k in ('reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion" /v EditionID') do (
  set Edition=%%k
)
Echo Windows_Edition_external=%Edition%
```

For example on my Windows 10 laptop it outputs:

```
Windows_Edition_external=Professional
```

And from within Puppet:

``` powershell
> puppet facts
...
    "kernel": "windows",
    "windows_edition_external": "Professional",
    "domain": "internal.local",
    "virtual": "physical",
...
```

### The custom fact

Firstly we need to create a boiler plate custom fact in the module by creating the following file `lib/facter/windowsedition.rb`

``` ruby
Facter.add('windows_edition_custom') do
  confine :osfamily => :windows
  setcode do
    'testvalue'
  end
end
```

This creates a custom fact called `windows_edition_custom` which has a value of `testvalue`.  Running facter on my laptop we see:

``` powershell
> puppet facts
...
    "windows_edition_custom": "testvalue",
    "clientcert": "glennsarti.internal.local",
    "clientversion": "5.0.0",
...
```

### Breaking down the custom fact code

So lets break down the boiler plate code:

``` ruby
Facter.add('windows_edition_custom') do
...
end
```

This instructs Facter to create a new fact called `windows_edition_custom`

``` ruby
  confine :osfamily => :windows
```

The confine statement instructs facter to only attempt resolution of this fact on Windows operating systems.

``` ruby
  setcode do
..
  end
```

`setcode` instructs facter to run the code block to resolve the fact's value

``` ruby
    'testvalue'
```

As this is just a demonstration, we are using a static string.  This is the code we'll change to output what the real value.

### Reading the registry in Puppet and ruby

You can access registry functions using the `Win32::Registry` namespace.  This is our new custom fact:

``` ruby
Facter.add('windows_edition_custom') do
  confine :osfamily => :windows
  setcode do
    value = nil
    Win32::Registry::HKEY_LOCAL_MACHINE.open('SOFTWARE\Microsoft\Windows NT\CurrentVersion') do |regkey|
      value = regkey['EditionID']
    end
    value
  end
end
```

So we've added five lines of code to read the registry.  Let's break these down too:

``` ruby
    value = nil
```

First we set value of the fact to nil.  We need to initialise the variable here otherwise, when we set it later inside the codeblock, its value will be lost due to [variable scoping](https://blog.codeship.com/a-deep-dive-into-ruby-scopes/)

Next we open the registry key `SOFTWARE\Microsoft\Windows NT\CurrentVersion`.  Not that unlike the batch file, it doesn't have the `HKLM` at the beginning.  This is because we're using the `HKEY_LOCAL_MACHINE` class, so adding that to the name is redundant.  By default the registry key is opened as Read Only and for 64bit access.

Next, once we have an open registry key, we get the registry value as a key in the `regkey` object, thus `regkey[EditionID']`.

Lastly we output the value for facter.  Ruby uses the output from the last line so we don't need an explicit `return` statement like you would in langauges like C#.

Now when we run the fact we get:

``` powershell
> puppet facts
...
    "windows_edition_custom": "Professional",
    "clientcert": "glennsarti.internal.local",
    "clientversion": "5.0.0",
...
```

Tada!, we've now convert a batch file based external registry fact, to a custom ruby fact in 10 lines.  But there's still a bit of cleaning up to do.

### Final touches

If the registy key or value does not exist, facter raises a warning, for example, if I change `value = regkey['EditionID']` to `value = regkey['EditionID_doesnotexist']` I see these errors output:

``` powershell
> puppet facts
...
Warning: Facter: Could not retrieve fact='windows_edition_custom', resolution='<anonymous>': The
system cannot find the file specified.
{
...
```

We could write some code to test for existance of registry keys and other techniques, but as this is just a fact we can safely just swallow any errors and not output the fact.  We can do this with a begin-rescue block.

``` ruby
Facter.add('windows_edition_custom') do
  confine :osfamily => :windows
  setcode do
    begin
      value = nil
      Win32::Registry::HKEY_LOCAL_MACHINE.open('SOFTWARE\Microsoft\Windows NT\CurrentVersion') do |regkey|
        value = regkey['EditionID']
      end
      value
    rescue
      nil
    end
  end
end
```

Much like the try-catch in PowerShell or C#, begin-rescue will catch the error and just output `nil` for the fact value if an error occurs.

## Writing a WMI based custom fact

### The external fact

For this example we'll convert a powershell file based external fact, to a ruby external fact.  This fact reads the `ChassisTypes` property of the [Win32_SystemEnclosure](https://msdn.microsoft.com/en-us/library/aa394474(v=vs.85).aspx) WMI Class.  This describes the type of physical enclosure for the computer, for example a Mini Tower, or in my case, a Portable device.

``` powershell
$enclosure = Get-WMIObject -Class Win32_SystemEnclosure | Select-Object -First 1

Write-Output "chassis_type_external=$($enclosure.ChassisTypes)"
```

For example on my Windows 10 laptop it outputs:

```
chassis_type_external=8
```

And from within Puppet:

``` powershell
> puppet facts
...
    "kernel": "windows",
    "chassis_type_external": "8",
    "domain": "internal.local",
    "virtual": "physical",
...
```

### The custom fact

Just like the last example, firstly we need to create a boiler plate custom fact in the module by creating the following file `lib/facter/chassistype.rb`

``` ruby
Facter.add('chassis_type_custom') do
  confine :osfamily => :windows
  setcode do
    'testvalue'
  end
end
```

### Accessing WMI in Puppet and ruby

We can access WMI using the `win32ole` and `winmgmts://` namespace.  If you ever used WMI in VBScript (yes I'm that old!) this may look familiar.

Note - I've already added the begin-rescue block

``` ruby
Facter.add('chassis_type_custom') do
  confine :osfamily => :windows
  setcode do
    begin
      require 'win32ole'
      wmi = WIN32OLE.connect("winmgmts:\\\\.\\root\\cimv2")
      enclosure = wmi.ExecQuery("SELECT * FROM Win32_SystemEnclosure").each.first

      enclosure.ChassisTypes
    rescue
    end
  end
end
```

So again, let's break this down:

``` ruby
      require 'win32ole'
```

Much like in PowerShell or C#, we need to import modules (or gems for ruby) into our code.  We do this with the `require` statement.  This enables us to use the WIN32OLE object on later lines.

``` ruby
      wmi = WIN32OLE.connect("winmgmts:\\\\.\\root\\cimv2")
```

We then connect to the local computer (local computer is denoted by the period) WMI, inside the `root\cimv2` scope.  Note that in ruby the backslash is an escape character so each backslash turns into a double backslash.  Although WMI can understand using forward slashes I had some ruby crashes in Ruby 2.3 using forward slashes.

``` ruby
      enclosure = wmi.ExecQuery("SELECT * FROM Win32_SystemEnclosure").each.first
```

Now that we have a WMI connection we can send it a standard WQL query for all Win32_SystemEnclosure objects.  As this returns an array, and there is only a single enclosure, we get the first element (`.each.first`) and discard anything else

``` ruby
      enclosure.ChassisTypes
```

And now we simply output the ChassisTypes parameter as the fact value.

This gives the following output:

``` powershell
> puppet facts
...
    "chassis_type_custom": [
      8
    ],
    "clientcert": "glennsarti.internal.local",
    "clientversion": "5.0.0",
...
```

Huh.  So the output is slightly different.  In external facts all output is considered a string.  However as we are now using WMI and custom ruby facts, we can properly understand data types.  Looking at the [MSDN](https://msdn.microsoft.com/en-us/library/aa394474(v=vs.85).aspx) documentation `ChassisTypes` is indeed an array type.

If this was ok for any dependant puppet code, we could leave the code as is.

However if you wanted just the first element we could use:

``` ruby
      enclosure.ChassisTypes.first
```

and this would output a single number, instead of a string:

``` powershell
> puppet facts
...
    "chassis_type_custom": 8,
    "clientcert": "glennsarti.internal.local",
    "clientversion": "5.0.0",
...
```

If you wanted it to be exactly like the external fact, we could then convert the integer into a string using `to_s`

``` ruby
      enclosure.ChassisTypes.first.to_s
```

and this would output a single number, instead of a string:

``` powershell
> puppet facts
...
    "chassis_type_custom": "8",
    "clientcert": "glennsarti.internal.local",
    "clientversion": "5.0.0",
...
```

## Final notes

### Structured Facts

[Structured facts](https://docs.puppet.com/facter/3.7/custom_facts.html#structured-facts) allow people to send more data than just a simple text string.  This is usually as encoded JSON or YAML data.  External facts have been able to provide structured facts e.g. A batch file that outputs pre-formatted JSON text, however this is not enabled for PowerShell yet (https://tickets.puppetlabs.com/browse/FACT-1653).

### `puppet facts` vs `facter`

In my examples above I was using the command `puppet facts` whereas most people would probably use `facter`.  This is mostly because I'm lazy.  By default just running facter won't evaluate custom facts in modules.  External facts are fine due to plugin sync.  By running `puppet facts`, puppet automatically generates the correct command ine to run facter with all of the custom facts paths.

One other reason is debugging.  In most modern Puppet installations facter is running as native facter which is a little harder to debug native ruby code in (not impossible though).  However when using Puppet as gem, as opposed to a puppet-agent installation, like I do when developing a module, it uses the Facter gem, which means I can use standard ruby debugging tools to help me out.

## Conclusion

I hope this blog post helps you see that writing simple custom facts isn't too daunting.  In fact, the hardest part is setting up a ruby development environment.  I came across a [blog post](https://blog.turtlesystems.co.uk/2017/06/28/Managing-Multiple-Rubies-on-Windows/) which explains setting up ruby, very similar to my environment.  Even though it's for Chef, it still works for Puppet too.

The source code for these examples is available on my [blog github repo](https://github.com/glennsarti/code-glennsarti.github.io/tree/master/customfacts).
