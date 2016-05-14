---
title: Tips for using the Puppet Powershell Module
excerpt: Tips for using the Puppet Powershell Module and how exit codes and errors affect resources
category:
  - blog
tags:
  - puppet
  - powershell
---

[Puppet](http://www.puppet.com) is a Configuration Management product which can be use to manage Infrastructure and applications.  It has great support for Windows, including a module on the [Puppet Forge](https://forge.puppet.com/puppetlabs/powershell) to run Powershell based commands.  There are some instructions in the README to help people use the module, but there a few traps for people starting out.

This post refers to version 1.x of the [Puppet Powershell module](https://forge.puppet.com/puppetlabs/powershell/1.0.6/readme)

## Powershell Exit Codes

The Puppet Powershell module makes use of the exit codes produced by powershell so it's important to understand how these codes are produced in Powershell.

The `Exit` command can be used to exit a powershell script and optionally return an exit code.

Unlike a lot scripting languages, Powershell has the concept of terminating and non-terminating errors ([More information](https://blogs.technet.microsoft.com/heyscriptingguy/2015/09/15/error-handling-two-types-of-errors/)).  A terminating error will stop the script whereas a non-terminating error will not.  This can cause some headaches for people that aren't expecting it!

Take this very simple batch file;
{% highlight powershell %}
@ECHO OFF
powershell.exe "& { exit 255}"
ECHO Errorlevel %ERRORLEVEL%
{% endhighlight %}
It produces
```
Errorlevel 255
```
This is what you would expect to happen.

What about if I don't explicitly set an exit code?
{% highlight powershell %}
@ECHO OFF
powershell.exe "& { }"
ECHO Errorlevel %ERRORLEVEL%
{% endhighlight %}
```
Errorlevel 0
```

So far so good.  So what happens if I raise an error (Division by Zero)?
{% highlight powershell %}
@ECHO OFF
powershell.exe "& { 1/0 }"
ECHO Errorlevel %ERRORLEVEL%
{% endhighlight %}
```
Attempted to divide by zero.
At line:1 char:5
+ & { 1/0 }
+     ~~~
    + CategoryInfo          : NotSpecified: (:) [], RuntimeException
    + FullyQualifiedErrorId : RuntimeException

Errorlevel 0
```
But the exit code is still zero?!?!?

What about if the script syntax was broken, like a missing bracket?
{% highlight powershell %}
@ECHO OFF
powershell.exe "& { (missing-bracket }"
ECHO Errorlevel %ERRORLEVEL%
{% endhighlight %}
```
At line:1 char:21
+ & { (missing-bracket }
+                     ~
Missing closing ')' in expression.
    + CategoryInfo          : ParserError: (:) [], ParentContainsErrorRecordException
    + FullyQualifiedErrorId : MissingEndParenthesisInExpression

Errorlevel 1
```
Now the exit code is no longer zero

What would happen if I changed the error from Non-Terminating to Terminating?
This behaviour is normally changed changing the `ErrorAction` parameter on Cmdlets or the global `$ErrorActionPreference` variable to `Stop`

{% highlight powershell %}
@ECHO OFF
powershell.exe "& { Get-Service -Name IDontExist }"
ECHO Errorlevel %ERRORLEVEL%

powershell.exe "& { Get-Service -Name IDontExist -ErrorAction Stop }"
ECHO Errorlevel %ERRORLEVEL%
{% endhighlight %}
```
Get-Service : Cannot find any service with service name 'IDontExist'.
At line:1 char:5
+ & { Get-Service -Name IDontExist }
+     ~~~~~~~~~~~~~~~~~~~~~~~~~~~~
    + CategoryInfo          : ObjectNotFound: (IDontExist:String) [Get-Service], ServiceCommandException
    + FullyQualifiedErrorId : NoServiceFoundForGivenName,Microsoft.PowerShell.Commands.GetServiceCommand

Errorlevel 0

Get-Service : Cannot find any service with service name 'IDontExist'.
At line:1 char:5
+ & { Get-Service -Name IDontExist -ErrorAction Stop }
+     ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
    + CategoryInfo          : ObjectNotFound: (IDontExist:String) [Get-Service], ServiceCommandException
    + FullyQualifiedErrorId : NoServiceFoundForGivenName,Microsoft.PowerShell.Commands.GetServiceCommand

Errorlevel 1
```
So the non-terminating error (without the ErrorAction parameter) produced a zero exit code while the terminating error (ErrorAction Stop) produced a non-zero exit code.

### What does this all mean?

If you depend on Powershell returning a meaningful exit code, you should use the exit command for all code branches.  This ensures Powershell returns the exit codes you expect.

{% highlight powershell %}
# Don't do this
If ($foo -eq 'bar') {
  exit 1
}
{% endhighlight %}

{% highlight powershell %}
# Do this instead
If ($foo -eq 'bar') {
  exit 1
}
exit 0
{% endhighlight %}

---
More information on [-ErrorAction](https://technet.microsoft.com/en-us/library/hh847884.aspx)

More infomration on [$ErrorActionPreference](https://technet.microsoft.com/en-us/library/hh847796.aspx)

## OnlyIf/Unless

The Puppet `exec` resource has [OnlyIf](http://somewhere) and [Unless](http://somewhere) attributes which can be used to limit when the command is invoked; e.g. Create this file only if it does not exist, or Start this windows service unless it's already running.

The onlyif parameter is defined as

> If this parameter is set, then this exec will only run if the command has an exit code of 0

Whereas the unless attribute is defined as

> If this parameter is set, then this exec will run unless the command has an exit code of 0

Using what we know about powershell exit codes a simple test manifest can be used to see what would happen for different scenarios;

{% highlight puppet %}
# OnlyIf tests
exec { 'onlyif check exit 0':
  command  => '"OnlyifExit0"',
  onlyif => 'exit 0',
  provider  => powershell,
} ->

exec { 'onlyif check exit 1':
  command  => '"OnlyifExit1"',
  onlyif => 'exit 1', 
  provider  => powershell,
} ->

exec { 'onlyif check noexit':
  command  => '"OnlyifNoExit"',
  onlyif => '',
  provider  => powershell,
} ->

exec { 'onlyif check non-term error':
  command  => '"OnlyifNonTerm"',
  onlyif => 'Get-Service -Name IDontExist;',
  provider  => powershell,
} ->

exec { 'onlyif check term error':
  command  => '"OnlyifTerm"',
  onlyif => 'Get-Service -Name IDontExist -ErrorAction Stop',
  provider  => powershell,
}  ->

# Unless tests
exec { 'unless check exit 0':
  command  => '"UnlessExit0"',
  unless => 'exit 0',
  provider  => powershell,
} ->

exec { 'unless check exit 1':
  command  => '"UnlessExit1"',
  unless => 'exit 1',
  provider  => powershell,
} ->

exec { 'unless check noexit':
  command  => '"UnlessNoExit"',
  unless => '',
  provider  => powershell,
} ->

exec { 'unless check non-term error':
  command  => '"UnlessNonTerm"',
  onlyif => 'Get-Service -Name IDontExist;',
  provider  => powershell,
} ->

exec { 'unless check term error':
  command  => '"UnlessTerm"',
  onlyif => 'Get-Service -Name IDontExist -ErrorAction Stop',
  provider  => powershell,
}
{% endhighlight %}
```
Notice: Compiled catalog for win-edson23cglf.localdomain in environment production in 0.16 seconds
Notice: /Stage[main]/Main/Exec[onlyif check exit 0]/returns: executed successfully
Notice: /Stage[main]/Main/Exec[onlyif check noexit]/returns: executed successfully
Notice: /Stage[main]/Main/Exec[onlyif check non-term error then command]/returns: executed successfully
Notice: /Stage[main]/Main/Exec[unless check exit 1]/returns: executed successfully
Notice: /Stage[main]/Main/Exec[unless check non-term error then command]/returns: executed successfully
Notice: Applied catalog in 18.28 secondss
```
So what did these tests show ...

* The explictly set exit code tests (`onlyif check exit 0`, `onlyif check exit 1`, `unless check exit 0` and `unless check exit 1`) behaved exactly as the documentation stated

* The no exit code set tests (`onlyif check noexit` and `unless check noexit`) behaved exactly as the documentation stated; as if it returned exit code 0

* The terminating error and non-terminating tests (`onlyif check non-term error`, `onlyif check term error`, `unless check non-term error` and `unless check term error`) did not behave correctly.  In all cases the exec resource was not run.  This isn't documented but kind of makes sense; If an error occurs in determining whether the exec resource should run, then the resource should not be executed.


So we've confirmed that the Powershell module is looking at errors and changes the exec behaviour in addition to exit codes.  So as one last test, what would happen if a non-terminating error was thrown but continued with other commands? 

{% highlight puppet %}
# OnlyIf tests
exec { 'onlyif check non-term error then command':
  command  => '"OnlyifNonTermThenCommand"',
  onlyif => 'Get-Service -Name IDontExist; Write-Host "Hello"',
  provider  => powershell,
} ->

# Unless tests
exec { 'unless check non-term error then command':
  command  => '"UnlessNonTermThenCommand"',
  onlyif => 'Get-Service -Name IDontExist; Write-Host "Hello"',
  provider  => powershell,
} ->
{% endhighlight %}
```
Notice: Compiled catalog for win-edson23cglf.localdomain in environment production in 0.16 seconds
Notice: /Stage[main]/Main/Exec[onlyif check non-term error then command]/returns: executed successfully
Notice: /Stage[main]/Main/Exec[unless check non-term error then command]/returns: executed successfully
Notice: Applied catalog in 18.28 secondss
```

* The non-terminating followed by a succesful command tests (`onlyif check non-term error then command` and `unless check non-term error then command`) result is just weird! In all cases the exec resource was executed!

### What does this all mean?

* Always use explicit exit codes

* Watch out for Terminating Errors as the exec resource will not run if they are thrown

{% highlight puppet %}
# Don't do this

# Start the FOO service unless already running or does not exist
exec { 'Start FOO':
  command  => 'Start-Service -Name "FOO"',
  unless => '$foo = Get-Service -Name "FOO";
             if ($foo.Status -eq "Running") { Exit 1 }',
  provider  => powershell,
}
{% endhighlight %}

{% highlight puppet %}
# Do this instead

# Start the FOO service unless already running or does not exist
exec { 'Start FOO':
  command  => 'Start-Service -Name "FOO"',
  unless => '$foo = Get-Service -Name "FOO";
             if ($foo.Status -eq "Running") { Exit 1 } else { Exit 0 }',
  provider  => powershell,
}
{% endhighlight %}

Without the `else { Exit 0 }` the exec resource will fail on computers without the FOO service.