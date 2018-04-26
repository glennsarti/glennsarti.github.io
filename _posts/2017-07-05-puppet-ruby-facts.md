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
modified: 2017-07-14
---

## Puppet, PowerShell and Facter

Puppet uses a tool called [Facter](https://docs.puppet.com/facter/latest/index.html) to gather system information during a Puppet run.

> Facter is Puppet's cross-platform system profiling library. It discovers and reports per-node facts, which are available in your Puppet manifests as variables.

There are some [core facts](https://docs.puppet.com/facter/latest/core_facts.html) which are processed on all operating systems, but two additional types of facts can be used to extend facter; External Facts and Custom Facts.

**External facts**

[External](https://docs.puppet.com/facter/latest/custom_facts.html#external-facts) facts provide a way to use arbitrary executables or scripts to generate facts as basic key / value pairs. If you’ve ever wanted to write a custom fact in Perl, C or PowerShell, this is how. Alternatively, external facts may contain static structured data in a JSON or YAML file.

**Custom Facts**

[Custom](https://docs.puppet.com/facter/latest/fact_overview.html) facts are written in Ruby and have more advanced features, allowing for programmatic confinement to specific environments.

Most people new to Facter will write PowerShell scripts as external facts.  However, there is a downside.  The execution time for PowerShell scripts can be a little slow as a result of the time required to start a new PowerShell process for each fact.  Another downside is that Windows will use file extensions to determine if a fact may be executed, while Unix based operating systems will look for the executable bit - it can be easy to forget these rules, especially when building cross-platform modules.  While there are some tricks to help stop this, it's easy for Windows scripts to log warnings and errors making it harder to figure out when real issues occur.

But the apparent learning curve to writing Ruby looks steep; if all you want to do is read a registry key and output the result, why should a Windows administrator have to learn Ruby?  Well, this blog post should help reduce the effort it takes to write custom facts.  Also, if you squint, the Ruby language looks a lot like (and in some cases operates similarly to) PowerShell.

The source code for these examples is available on my [blog github repo](https://github.com/glennsarti/code-glennsarti.github.io/tree/master/customfacts).

## Writing a registry based custom fact

### The external fact

For this example we'll convert a batch file based external fact to a Ruby external fact.  This fact reads the `EditionID` of the operating system from the registry and then populates a fact called `Windows_Edition`.

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

The `confine` statement instructs Facter to only attempt resolution of this fact on Windows operating systems.

``` ruby
  setcode do
..
  end
```

`setcode` instructs Facter to run the code block to resolve the fact's value

``` ruby
    'testvalue'
```

As this is just a demonstration, we are using a static string.  This is the code we'll subsequently change to output a real value.

### Reading the registry in Puppet and Ruby

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

First we set value of the fact to nil.  We need to initialise the variable here, otherwise when its value is later set inside the codeblock, its value will be lost due to [variable scoping](https://blog.codeship.com/a-deep-dive-into-ruby-scopes/)

Next we open the registry key `SOFTWARE\Microsoft\Windows NT\CurrentVersion`.  Note that unlike the batch file, it doesn't have the `HKLM` at the beginning.  This is because we're using the `HKEY_LOCAL_MACHINE` class, so adding that to the name is redundant.  By default the registry key is opened as Read Only and for 64-bit access.

Next, once we have an open registry key, we get the registry value as a key in the `regkey` object, thus `regkey['EditionID']`.

Lastly, we output the value for Facter.  Ruby uses the output from the last line so we don't need an explicit `return` statement like you would in langauges like C#.

When we run the updated fact we get:

``` powershell
> puppet facts
...
    "windows_edition_custom": "Professional",
    "clientcert": "glennsarti.internal.local",
    "clientversion": "5.0.0",
...
```

Tada!, we've now converted a batch file based external registry fact, to a custom Ruby fact in 10 lines.  But there's still a bit of cleaning up to do.

### Final touches

If the registy key or value does not exist, Facter raises a warning. For example, if I change `value = regkey['EditionID']` to `value = regkey['EditionID_doesnotexist']` I see these errors output:

``` powershell
> puppet facts
...
Warning: Facter: Could not retrieve fact='windows_edition_custom', resolution='<anonymous>': The
system cannot find the file specified.
{
...
```

We could write some code to test for existence of registry keys, but as this is just a fact we can simply swallow any errors and not output the fact.  We can do this with a `begin` / `rescue` block.

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

Much like the `try` / `catch` in PowerShell or C#, `begin` / `rescue` will catch the error and just output `nil` for the fact value if an error occurs.

## Writing a WMI based custom fact

### The external fact

For this example we'll convert a PowerShell file based external fact, to a Ruby external fact.  This fact reads the `ChassisTypes` property of the [Win32_SystemEnclosure](https://msdn.microsoft.com/en-us/library/aa394474\(v=vs.85\).aspx) WMI Class.  This describes the type of physical enclosure for the computer, for example a Mini Tower, or in my case, a Portable device.

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

Just like the last example, we start with a boilerplate custom fact in the module by creating the following file `lib/facter/chassistype.rb`

``` ruby
Facter.add('chassis_type_custom') do
  confine :osfamily => :windows
  setcode do
    'testvalue'
  end
end
```

### Accessing WMI in Puppet and ruby

We can access WMI using the `WIN32OLE` Ruby class and `winmgmts://` WMI namespace.  If you ever used WMI in VBScript (yes I'm that old!) this may look familiar.

Note - I've already added the `begin` / `rescue` block:

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

Much like in PowerShell or C#, we need to import modules (or gems for Ruby) into our code.  We do this with the `require` statement.  This enables us to use the `WIN32OLE` object on later lines.

``` ruby
      wmi = WIN32OLE.connect("winmgmts:\\\\.\\root\\cimv2")
```

We then connect to the local computer (local computer is denoted by the period) WMI, inside the `root\cimv2` scope.  Note that in Ruby the backslash is an escape character so each backslash must be escaped as a double backslash.  Although WMI can understand using forward slashes I had some Ruby crashes in Ruby 2.3 using forward slashes.

``` ruby
      enclosure = wmi.ExecQuery("SELECT * FROM Win32_SystemEnclosure").each.first
```

Now that we have a WMI connection we can send it a standard WQL query for all Win32_SystemEnclosure objects.  As this returns an array, and there is only a single enclosure, we get the first element (`.each.first`) and discard anything else

``` ruby
      enclosure.ChassisTypes
```

And now we simply output the `ChassisTypes` parameter as the fact value.

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

Huh.  So the output is slightly different.  In external executable facts all output is considered a string.  However as we are now using WMI and custom ruby facts, we can properly understand data types.  Looking at the [MSDN](https://msdn.microsoft.com/en-us/library/aa394474(v=vs.85).aspx) documentation `ChassisTypes` is indeed an array type.

If this was ok for any dependent Puppet code, we could leave the code as is.

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

and this would output a single string , instead of a number:

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

[Structured facts](https://docs.puppet.com/facter/3.7/custom_facts.html#structured-facts) allow people to send more data than just a simple text string.  This is usually as encoded JSON or YAML data.  External facts have been able to provide structured facts, for instance using a batch file to output pre-formatted JSON text, but this is not yet enabled for PowerShell (https://tickets.puppetlabs.com/browse/FACT-1653).

### `puppet facts` vs `facter`

In my examples above I was using the command `puppet facts` whereas most people would probably use `facter`.  This is mostly because I'm lazy.  By default just running Facter (`facter`) won't evaluate custom facts in modules.  External facts are fine due to pluginsync.  By running `puppet facts`, Puppet automatically runs Facter with all of the custom facts paths loaded.  Note, `facter -p` works but is deprecated in favour of `puppet facts`

One other reason is debugging.  In most modern Puppet installations Facter is running as native Facter which can make debugging native Ruby code trickier (though not impossible).  However, when using Puppet as gem instead of installing the puppet-agent package, common during module development, it uses the Facter gem. The Facter gem allows for using standard Ruby debugging tools to help me out.

## Conclusion

I hope this blog post helps you see that writing simple custom facts isn't too daunting.  In fact, the hardest part is setting up a Ruby development environment.  I came across a [blog post](https://blog.turtlesystems.co.uk/2017/06/28/Managing-Multiple-Rubies-on-Windows/) which explains setting up Ruby, very similar to my environment.  Even though it's for Chef, it still works for Puppet too.

The source code for these examples is available on my [blog github repo](https://github.com/glennsarti/code-glennsarti.github.io/tree/master/customfacts).

Thanks to Ethan Brown ([@ethanjbrown](https://twitter.com/ethanjbrown)) for editing.
