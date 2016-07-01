---
title: Update - Tips for using the Puppet Powershell Module v2.0.0
excerpt: Updated tips for using the Puppet Powershell Module, version 2.0.0
category:
  - blog
header:
  image: teaser-powershell.png
  overlay_image: header-powershell.png
  teaser: teaser-powershell.png
tags:
  - puppet
  - powershell
---

In a [previous](/blog/powershell-puppet-module-exit-codes/) post we looked tips for using the Powershell Module.  On May 18th, a new version has been [released](https://forge.puppet.com/puppetlabs/powershell/2.0.0/readme) which has made some huge performance improvements, but also fixed some bugs with error handling.

## Performance Improvements

To give you some idea of the improvement I used the test manifest from [this](/blog/powershell-puppet-module-exit-codes/) post and timed how long it took to run.  Of course, I used Powershell to do it..

``` powershell
#### OLD MODULE
Write-Host "*** OLD Module"
$oldModuleTime = Measure-Command { & puppet apply powershell.pp --modulepath C:\blog\old-module }

$oldModuleTime.TotalSeconds

#### NEW MODULE
Write-Host "*** NEW Module"
$newModuleTime = Measure-Command { & puppet apply powershell.pp --modulepath C:\blog\new-module }

$newModuleTime.TotalSeconds

#### BASELINE
Write-Host "*** Baseline"
$baselineTime = Measure-Command { & puppet apply -e "#" --modulepath C:\blog\old-module }

$baselineTime.TotalSeconds

### Back of Napkin improvement
$oldModuleTime = $oldModuleTime - $baselineTime
$newModuleTime = $newModuleTime - $baselineTime
Write-Host "*** Improvement"
Write-Host ($oldModuleTime.TotalMilliseconds / $newModuleTime.TotalMilliseconds).ToString("###%")
```

```
*** OLD Module
Error: Write-error "Hello" returned 1 instead of one of [0]
Error: /Stage[main]/Main/Exec[write-error]/returns: change from notrun to 0 failed: Write-error "Hello" returned 1 instead of one of [0]
32.1743243
*** NEW Module
22.2315182
*** Baseline
12.4175615
*** Improvement
201%
```

So the old module took 32 seconds while the new module took 22 seconds, but that isn't the whole story.  The baseline measures how long Puppet takes to just start up, so it's not even executing powershell then.  Taking the baseline time away, the old module took 20 seconds (32 - 12) and the new module took 10 seconds (22 - 12).

So the new module is **2x** faster than the old one!  Also, there's less disk IO and there's no temporary file on disk, so that security concern is gone.

Obviously the performance improvement you see will change depending on your scenario, but in essence, the more you've used the Powershell resource in your manifests, the greater the improvement.


## Error Handling

The error handling has changed slightly in the new module, so lets see what happens when we run the previous test manifest.  I've also included the non-terminating error with extra command tests too.

Old Module

```
Notice: Compiled catalog for win-edson23cglf.localdomain in environment production in 0.17 seconds
Notice: /Stage[main]/Main/Exec[onlyif check exit 0]/returns: executed successfully
Notice: /Stage[main]/Main/Exec[onlyif check noexit]/returns: executed successfully
Notice: /Stage[main]/Main/Exec[onlyif check non-term error then command]/returns: executed successfully
Notice: /Stage[main]/Main/Exec[unless check exit 1]/returns: executed successfully
Notice: /Stage[main]/Main/Exec[unless check non-term error]/returns: executed successfully
Notice: /Stage[main]/Main/Exec[unless check term error]/returns: executed successfully
```

New Module

```
Notice: Compiled catalog for win-edson23cglf.localdomain in environment production in 0.22 seconds
Notice: /Stage[main]/Main/Exec[onlyif check exit 0]/returns: executed successfully
Notice: /Stage[main]/Main/Exec[onlyif check noexit]/returns: executed successfully
Notice: /Stage[main]/Main/Exec[onlyif check non-term error]/returns: executed successfully
Notice: /Stage[main]/Main/Exec[onlyif check non-term error then command]/returns: executed successfully
Notice: /Stage[main]/Main/Exec[unless check exit 1]/returns: executed successfully
Notice: /Stage[main]/Main/Exec[unless check term error]/returns: executed successfully
```

So what's changed?

* The `onlyif check non-term error` test now runs the exec resource.  This is same behaviour as `onlyif check non-term error then command` and now treats all non terminating errors the same i.e. exit code zero.

* The `unless check non-term error` no longer runs the exec resource.  This is because the exit code is zero for non-terminating errors.
