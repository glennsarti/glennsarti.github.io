---
title: How we propelled PowerShell at Puppet
excerpt: Repost from a blog article I wrote for puppet.com
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
  - powershell
modified: 2017-07-14
---

Originally posted on [puppet.com](https://puppet.com/blog/how-propelled-powershell-at-puppet).

---

**Editor’s note:** ***Glenn wrote this blog post from his community presentation at the [PowerShell + DevOps Global Summit](https://eventloom.com/event/home/summit2017) held in April 2017. You'll find information about the sessions and community demos here: [https://github.com/devops-collective-inc/summit-materials](https://github.com/devops-collective-inc/summit-materials).***

The Puppet PowerShell module is one of the most popular modules for Windows administrators on the Puppet Forge. How popular, you ask?

* 2014 - 7,762 downloads
* 2015 - 79,241 downloads
* 2016 - 10,476,787 downloads!!

One reason for the large increase in downloads is the speed improvements we made to the module. (Of course, it was also the first new version for a while, and much anticipated.) In this blog post, I'll walk through the improvements we made to the PowerShell module, and how we increased its speed.

## Version 1.0.6

Previously, the PowerShell module would start a new PowerShell process for each command it executed. For a single `exec` resource, that could mean two separate PowerShell executions when specifying an  `onlyif` or `unless`. For example:

``` puppet
exec { 'change-password':
  command   => '... <powershell> ...',
  onlyif    => '... <powershell> ...',
  provider  => powershell,
}

exec { 'rename-guest':
  command   => '... <powershell> ...',
  onlyif    => '... <powershell> ...',
  provider  => powershell,
}
```

These two simple exec resources call PowerShell four times in total, because each `onlyif` and `command` statement launches a new PowerShell process. While this worked quite well, it was slow when you had many exec resources.

![puppet exec](https://puppet.com/sites/default/files/2017-07/version1.0.6.gif)

## Version 2.0.0: speed!

Version 2.0.0 was the first attempt to speed up the PowerShell module. Instead of creating a new PowerShell session every time, we execute PowerShell [runspaces](https://blogs.technet.microsoft.com/heyscriptingguy/2015/11/26/beginning-use-of-powershell-runspaces-part-1/) within a single hosted PowerShell process. Runspaces are faster to create, as they are essentially a new thread of execution rather than an entirely new process.

PowerShell runspaces, as we know them now, started in PowerShell 2.0, and are a foundational part of how PowerShell works. From the [MSDN documentation](https://msdn.microsoft.com/en-us/library/system.management.automation.runspaces.runspace.aspx):

> The user at the command line may not necessarily realize that the commands that Windows PowerShell runs are being processed within a runspace. From the command-line user’s point of view, commands are run in a Windows PowerShell session. From a host application’s point of view, commands are run within a runspace.

Runspaces provide a level of isolation, as this is the container for most variables and functions. However, variables created in the global scope, and environment variables created in the process scope, can still bleed across runspaces. The PowerShell module uses a custom PowerShell host that ensures this data is cleared before each Puppet resource executes.

Isolation is important, because changes that are made to things like environment variables, script variables, or functions should not be visible across multiple invocations. For example, if you created a variable called `$SecurePassword` in the `change-password` resource above and it contained an important password, you would not want it to be seen in the `rename-guest` resource. Each PowerShell statement (`onlyif`, `unless` or `command`) should be self-contained — isolated so they can't contaminate each other. 


Isolation also protects against errors or badly written PowerShell because only the runspace will be affected, not the entire PowerShell process.

The first execution of a PowerShell statement starts PowerShell.exe. However from then on, each statement runs in a PowerShell runspace, as seen below. (RS is a PowerShell runspace.)

![faster puppet powershell](https://puppet.com/sites/default/files/2017-07/version2.0.0.gif)

See the **Learn more** section at the end of this post for more information about runspaces.

### How much faster?

How much faster the PowerShell module will run depends very much on your individual circumstances. During the development process, we used a simple manifest to ensure everything was working. It was composed of more than 20 one-line PowerShell scripts like this:

``` puppet
...
exec { 'WriteOutput':
  command   => 'Write-Output "Should see this!"',
  provider  => powershell,
}

exec { 'WriteHost':
  command   => 'Write-Host "Should see this!"',
  provider  => powershell,
}
...
```

Using this new technique, we saw a speed improvement of 200 percent when running PowerShell statements. Fantastic!


Why did we choose these trivial test cases? Two reasons: First, we wanted to make sure we maintained the same behavior from Version 1.0.6.  Secondly, we were focused on improving the startup speed of the PowerShell statements, so we kept the actual statements small, letting us easily observe any differences we made.

## Version 2.0.2: more speed

In version 2.0.2, we managed to increase the speed of the module again. We noticed the PowerShell code for starting the PowerShell runspaces was being executed too often. By moving some of the runspace startup code from the initialization logic into the hosting PowerShell process, we could reduce the time it took to start a runspace.

Using the same testing harness as Version 2.0.0, we went from the previous 200 percent increase to a 230 percent speed increase.

## Windows 10 Anniversary Update and Windows Server 2016

Not long after the Windows 10 Anniversary Update was released, we noticed that the PowerShell module was no longer working. After much investigation, we found that the new version of PowerShell (5.1.14394.100) had a few changes made to help with cross-platform support so that PowerShell could be run on non-Windows systems. However, it broke our PowerShell module.

The good news was that, at this point, PowerShell was open source so we could both debug the problem and submit feedback with suggested fixes. We logged two issues: ([Issue #2068](https://github.com/PowerShell/PowerShell/issues/2068) and [Issue #2071](https://github.com/PowerShell/PowerShell/issues/2071)). The first was a change in how curly braces (`{ }`) were being parsed, and the second was that STDIN parsing was not working correctly.

The Microsoft PowerShell team was very responsive, and a fix was quickly put in for PowerShell 6. But PowerShell 5 was still affected. This included Windows 10, and the impending release of Windows Server 2016.

The Puppet JIRA issue [MODULES-3690](https://tickets.puppetlabs.com/browse/MODULES-3690) has more detailed information about the errors.

## Version 2.1.0: binary pipes

After discovering that the problem with the newer version of PowerShell was the handling of text via STDIN, we changed to using [named pipes](https://msdn.microsoft.com/en-us/library/windows/desktop/aa365590(v=vs.85).aspx). Interestingly, STDIN and STDOUT redirection is actually implemented with named pipes, but by using our own named pipe server and client, we could finely control the connection. Apart from solving the PowerShell 5.1 issue, it had added benefits:

* STDIN/STDOUT redirection can be tricky on Windows with different user interface languages and code pages. By using named pipes, we could transfer raw bytes instead of text strings, and so preserve the original text from PowerShell in Puppet.

* Previously, we had to put in a workaround for how PowerShell managed STDOUT, which meant encoding the text as Base64 to circumvent string formatting issues when passing data between Ruby and PowerShell. Now that we had binary named pipes, we no longer had to do this expensive encoding and decoding. This resulted in a speed increase and less memory overhead. For example, take a look at  the following resource:

``` puppet
exec { ‘test‘:
  command  => ‘Get-Service | Format-List‘,
  provider => powershell
}
```


Previously, this resource would take around 6.5 seconds to complete. But with the binary named pipes, it now takes only about 2.5 seconds. So if you are outputting log files or large amounts of text to help when debugging, this speed increase will be quite noticeable.

* We could now trap more streams from PowerShell, not just STDOUT, but warning, verbose, information and so on.

## Conclusion

By closely looking at how we were using PowerShell, we were able to make all these improvements, which added up to much better performance for our PowerShell module. We're happy that so many people have downloaded it and are getting good use from it. If you have any feedback for us, please share it on the #windows channel on Puppet Community Slack channel. If you have any issues, we want to know, so please [create a JIRA ticket](https://tickets.puppet.com/browse/MODULES/component/12015).

*Glenn Sarti is a senior software engineer at Puppet.*

## Learn more

* Check out the Puppet PowerShell module on the Puppet Forge: [https://forge.puppet.com/puppetlabs/powershell](https://forge.puppet.com/puppetlabs/powershell). Note: It's a Puppet Supported module, which means it's included in your professional support package if you are using Puppet Enterprise.
* White paper: [Managing Windows with Puppet Enterprise](https://puppet.com/resources/whitepaper/managing-windows-with-puppet-enterprise)
*  [Tips for using the PowerShell module](https://puppet.com/blog/tips-using-puppet-powershell-module)
* Remember to join the #windows room on the Puppet Community Slack channel to learn more about Puppet on Windows.
* Four-part series: [Beginning use of PowerShell Runspaces](https://blogs.technet.microsoft.com/heyscriptingguy/2015/11/26/beginning-use-of-powershell-runspaces-part-1/) from [the Scripting Guys](https://social.technet.microsoft.com/profile/The+Scripting+Guys).
* [MSDN documentation on the runspace class](https://msdn.microsoft.com/en-us/library/system.management.automation.runspaces.runspace.aspx)
* [Writing a Windows PowerShell Host Application](https://msdn.microsoft.com/en-us/library/ee706563(v=vs.85).aspx) -  includes managing a runspace.

---

Originally posted on [puppet.com](https://puppet.com/blog/how-propelled-powershell-at-puppet).
