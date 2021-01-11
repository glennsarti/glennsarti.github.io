---
title: Dynamic Test Matrices
excerpt: "Creating GitHub Action test matrices on the fly"
category:
  - blog
header:
  overlay_image: /images/header-matrix.png
  overlay_filter: 0.5
  caption: "Picture credit: [**markusspiske**](https://unsplash.com/@markusspiske)"
  teaser: /images/teaser-matrix.png
tags:
  - powershell
  - github
  - actions
read_time: true
toc: true
---

# The problem

I was working on the [Puppet Editor Services](https://github.com/puppetlabs/puppet-editor-services) project, moving the automated CI pipelines from Travis and AppVeyor over to GitHub Actions. The project uses different combinations of Operating Systems, Ruby versions and Puppet versions to test one. This was achieved using [Travis Build Matrix](https://docs.travis-ci.com/user/build-matrix/) and [AppVeyor Build Matrix](https://www.appveyor.com/docs/build-configuration/#build-matrix). But when I tried to the same in [GitHub Actions Matrix](https://docs.github.com/en/free-pro-team@latest/actions/reference/workflow-syntax-for-github-actions#jobsjob_idstrategymatrix) it just didn't work the same 😢.

In short - You can't add matrix entries that share the same keys. Instead you have to explicitly list out all combinations

In my case, I had the following test cases in the matrix

``` yaml
  os: ['windows-latest', 'ubuntu-latest']
  ruby: ['2.4', '2.5', '2.7']
  command: ['... run all tests ...']
```

This would run the tests for all the combinations of Operating System and Ruby Version. This worked great, however I also needed to add a specific Puppet Version, and therefore Ruby Version, test, as there is a regression in Puppet that needed to be tested. Also, we only needed to run a subset of the tests, not the whole suite. So the matrix now looked like:

``` yaml
  os: ['windows-latest', 'ubuntu-latest']
  ruby: ['2.4', '2.5', '2.7']
  command: ['... run all tests ...']
  include:
    - os: 'ubuntu-latest'
      ruby : '2.5'
      puppet_version: '5.1.0'
      command : ['... only run unit tests ...']
```

However GA (GitHub Actions) would then not run the "All Tests" command for Ruby 2.5 on ubuntu-latest. This was because the matrix was not adding, but overwriting. A quick search on this and I discovered this a known "thing" with the matrix feature in GA. I tried to re-architect the matrix and rethink how I would do the testing but nothing I came up with really worked or was just too horrible to maintain.

## Partial solution

The solution offered by GA was to not use a matrix at all, but define all the combinations explicitly. This meant turning my example 3 line matrix into a 24 line hardcoded list. This didn't take into account the other tests that I hadn't mentioned yet! I tried YAML anchors to reduce the copying of information but GA doesn't support YAML anchors.

Sure this _could_ work but it seemed like hard-coding is a bad way to go. So more searching and I came across a post about [sharing matrix information between jobs](https://github.community/t/how-to-share-matrix-between-jobs/128595), in particular this YAML example:

{% raw %}
``` yaml
job1:
  runs-on: ubuntu-latest
  outputs:
    matrix: ${{ steps.set-matrix.outputs.matrix }}
  steps:
  - id: set-matrix
    run: |
      echo "::set-output name=matrix::[{\"go\":\"1.13\",\"commit\":\"v1.0.0\"},{\"go\":\"1.14\",\"commit\":\"v1.2.0\"}]"

builder:
  needs: job1
  runs-on: ubuntu-latest
  strategy:
    matrix:
      cfg: ${{fromJson(needs.job1.outputs.matrix)}}
  steps:
  - run: |
      echo bin-${{ matrix.cfg.go }}-${{ matrix.cfg.commit }}
```
{% endraw %}

So what's going on here? Well this line looks very interesting:

{% raw %}
``` yaml
  matrix:
    cfg: ${{fromJson(needs.job1.outputs.matrix)}}
```
{% endraw %}

Instead of the matrix configuration being defined in YAML, it was consuming the JSON output from another job?!?! So the execution of the job looked like:

* `job1` starts
  * Runs the step `set-matrix`
  * The `set-matrix` step outputs a step variable called `matrix`. This is a compact JSON string.
    See the [set-output](https://docs.github.com/en/free-pro-team@latest/actions/reference/workflow-commands-for-github-actions#setting-an-output-parameter) documentation for more information about how to set output parameters
* The `builder` job starts after `job1` completes
  * The matrix configuration is deserialised from the JSON string value `needs.job1.outputs.matrix`
  * The steps are then run for each entry on the matrix

The example uses the `matrix.cfg` setting, but GA now supports using `matrix.include` directly. You'll see this later in this post
{: .notice--warning}

This means we can use an arbitrary JSON file to configure the matrix! But so what; Having a hardcoded JSON string that's read by a job is no different than hardcoding the YAML in the first place!

... But

... What if the JSON string wasn't hardcoded?

... What if the JSON string was created on the fly? by a PowerShell script? 🤔

## Crafting Test Matrices

So lets start with a fresh GA Workflow called `dynamic-test-matrix`.

{% raw %}
``` yaml
name: dynamic-test-matrix

on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main

jobs:
  matrix:
    name: Generate test matrix
    runs-on: ubuntu-latest
    outputs:
      matrix-json: ${{ steps.set-matrix.outputs.matrix }}
    steps:
      - uses: actions/checkout@v2
      - id: set-matrix
        shell: pwsh
        # Use a small PowerShell script to generate the test matrix
        run: "& .github/workflows/create-test-matrix.ps1"

  run-matrix:
    needs: [matrix]
    strategy:
      fail-fast: false
      matrix:
        include: ${{ fromJson(needs.matrix.outputs.matrix-json) }}
    name: "${{ matrix.job_name }}"
    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/checkout@v2
      - name: Run Command
        shell: pwsh
        run: |
          Write-Host "Run '${{ matrix.command }}'"
```
{% endraw %}

Just like the example before there are two jobs:

* `matrix` : Runs a PowerShell script to create the test matrix
* `run-matrix` : Pretends to the run the command for each item in the matrix

### The matrices creation job

Let's look the `matrix` job:

``` yaml
    name: Generate test matrix
```
A nice friendly name in the UI

``` yaml
    runs-on: ubuntu-latest
```
The job will run on an Ubuntu based runner. Why Ubuntu instead of Windows? There is no reason why it has to be Windows specific so why not make it a cross platform script.

{% raw %}
``` yaml
    outputs:
      matrix-json: ${{ steps.set-matrix.outputs.matrix }}
```
{% endraw %}
This instructs GA that it will output a Job parameter called `matrix-json`, and that its value comes from the step output `matrix`, from the step called `set-matrix`.

Next comes the steps for this job

``` yaml
    steps:
      - uses: actions/checkout@v2
```
The steps need the PowerShell script to run, so we do need to checkout the project source code first

``` yaml
      - id: set-matrix
        shell: pwsh
        # Use a small PowerShell script to generate the test matrix
        run: "& .github/workflows/create-test-matrix.ps1"
```
The `set-matrix` step the runs the `create-test-matrix.ps1` PowerShell Script (We'll get to the script soon). Note that the `id` is important here as this is name used in the job output above.

### Using the matrices

Let's look the `run-matrix` job:

```yaml
    needs: [matrix]
```
This job (`run-matrix`) needs the output of the `matrix` job, so this instructs GA to wait until the `matrix` job completes.

{% raw %}
```yaml
    strategy:
      fail-fast: false
```
{% endraw %}

For the purposes of this blog post, I want all of the test cells to run even if other ones have failed. The [GA Documentation](https://docs.github.com/en/free-pro-team@latest/actions/reference/workflow-syntax-for-github-actions#jobsjob_idstrategyfail-fast) has more examples of the matrix configuration

{% raw %}
``` yaml
      matrix:
        include: ${{ fromJson(needs.matrix.outputs.matrix-json) }}
```
{% endraw %}
This is where the magic happens, where we take the output from matrix job, to configure the matrix for this job. This is similar to the "Sharing matrix information between jobs" post, where it deserialises the JSON string from the `matrix` job output parameter called `matrix-json`.

Note that I'm using `include` instead of `cfg` like the post above. This makes it easier to reference matrix items later in the steps.  For example, instead of `matrix.cfg.os` we can just use `matrix.os`.

{% raw %}
``` yaml
    name: "${{ matrix.job_name }}"
    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/checkout@v2
      - name: Run Command
        shell: pwsh
        run: |
          Write-Host "Run '${{ matrix.command }}'"
```
{% endraw %}

And then finally the steps for the job.

* `matrix.job_name` is used to dynamically change the friendly name of the job
* `matrix.os` is used to change which job runner is used
* `matrix.command` is used to show what PowerShell command could be run

This example won't actually run any PowerShell commands but you can make GA run a PSake command, or PSBuild command, or run other PowerShell scripts in your project.
{: .notice--warning}

### Creating matrices in PowerShell

Here's an example script that will output a two cell matrix: One cell for Windows and one for Ubuntu.

This would be saved as `.github/workflows/create-test-matrix.ps1`. If you change the name of this script, make sure you also change the GA workflow to use the new name too.

``` powershell
$Jobs = @()

@('ubuntu-latest', 'windows-latest') | ForEach-Object {
  $Jobs += @{
    job_name = "Run $_ jobs"
    os = $_
    command = "$_ command"
  }
}

Write-Host "::set-output name=matrix::$($Jobs | ConvertTo-JSON -Compress))"
```

So what's going on here:

``` powershell
$Jobs = @()
```
We store the Job information in the `$Jobs` variable. Initially we have no jobs

```powershell
@('ubuntu-latest', 'windows-latest') | ForEach-Object {
```

Instead of hardcoding, we can use loops and enumeration to create matrix cells

```powershell
  $Jobs += @{
    job_name = "Run $_ jobs"
    os = $_
    command = "$_ command"
  }
```

To create a matrix cell we add a HashTable to the `$Jobs` array. Each key in the HashTable appears as a matrix variable in the GA Workflow.  In this example we are setting three keys; `job_name`, `os` and `command`. These are then used in the Workflow as `matrix.job_name`, `matrix.os` and `matrix.command` respectively. And each matrix cell does _not_ has to have the same keys. It's completely up to you to what each matrix cell has.

``` powershell
Write-Host "::set-output name=matrix::$($Jobs | ConvertTo-JSON -Compress))"
```

And then lastly we use the `set-output` magic text and convert the Jobs into a JSON string. Note the use of `-Compress` here. GA doesn't allow line breaks in the output so compress is used to create a JSON string on a single line.

When you run this script you get the following output:

``` text
::set-output name=matrix::[{"job_name":"Run ubuntu-latest jobs","command":"ubuntu-latest command","os":"ubuntu-latest"},{"job_name":"Run windows-latest jobs","command":"windows-latest command","os":"windows-latest"}])
```

Which, let's be honest is hard to read. Let's add a script parameter called `Raw` which will output the JSON in a readable way for humans

``` powershell
param(
  [Switch]$Raw,
)
$Jobs = @()

@('ubuntu-latest', 'windows-latest') | ForEach-Object {
  $Jobs += @{
    job_name = "Run $_ jobs"
    os = $_
    command = "$_ command"
  }
}

if ($Raw) {
  Write-Host ($Jobs | ConvertTo-JSON)
} else {
  # Output the result for consumption by GitHub Actions
  Write-Host "::set-output name=matrix::$($Jobs | ConvertTo-JSON -Compress))"
}
```

So now running `.github/workflows/create-test-matrix.ps1 -Raw`

``` text
[
  {
    "job_name": "Run ubuntu-latest jobs",
    "command": "ubuntu-latest command",
    "os": "ubuntu-latest"
  },
  {
    "job_name": "Run windows-latest jobs",
    "command": "windows-latest command",
    "os": "windows-latest"
  }
]
```

## Seeing the GitHub Action in action

![GitHub Action Workflow : Animated gif of the workflow running]({{ site.url }}/blog-images/dynamic-test-matrix-01.gif){: .align-center}

* Running the workflow we see first that the `Generate test matrix` job is running but there are not yet any subsequent jobs

* Once the generation job is complete, two new jobs appear. These are the two jobs we specified in our PowerShell script: `Run ubuntu-latest jobs` and `Run windows-latest jobs`

* When we look at the output of these commands we can see that the Ubuntu job has `Write-Host "Run 'ubuntu-latest command'` and the Windows job has `Write-Host "Run 'windows-latest command'`, just like we specified in our PowerShell script

## Back to the original problem =...

Back to Puppet Editor Services ... Now I could create a [PowerShell script](https://github.com/puppetlabs/puppet-editor-services/blob/main/.github/workflows/create-test-matrix.ps1) which created a JSON string with all of the test cases I needed (12 in total), I could easily make out what each cell matrix did, and I could easily add and remove test cases in the future.

![GitHub Action Workflow : Puppet Editor Services output from main]({{ site.url }}/blog-images/dynamic-test-matrix-02.png){: .align-center}

All of the code for this is in [Pull Request 288 of the Puppet Editors Services project](https://github.com/puppetlabs/puppet-editor-services/pull/288).

# Going further

Now that we are using PowerShell to generate the matrix, it opens up more opportunities:

You could add some tests if the person who raised the Pull Request had the name 'glennsarti'

``` powershell
if ($ENV:GITHUB_ACTOR -eq 'glennsarti') {
  $Jobs += @{
    # ...
  }
}
```

See the [documentation](https://docs.github.com/en/free-pro-team@latest/actions/reference/environment-variables#default-environment-variables) for the full list of GA Environment Variables
{: .notice--warning}

Or...

## Context aware testing

What if we could detect which files were being changed in a Pull Request and then change the testing. For example:

Let's say we had a PowerShell module which included documentation. If a Pull Request was **ONLY** changing the documentation files then there'd be no need to run PowerShell script tests. And vice versa.

Fortunately git can help us here. We can use `git diff --name-only` to list all of the files that are affected.

``` powershell
param(
  [Switch]$Raw,
  [String]$FromRef
)
$Jobs = @()

$TestModule = $false
$TestDocs = $false

if (![String]::IsNullOrWhiteSpace($FromRef)) {
  (& git diff --name-only $FromRef...HEAD) | ForEach-Object {
    if ($_ -like 'src/*') { $TestModule = $true }
    if ($_ -like 'docs/*') { $TestDocs = $true }
  }
}

# Make sure we test something
if (!$TestModule -and !$TestDocs) {
  $TestModule = $true
  $TestDocs = $true
}

@('ubuntu-latest', 'windows-latest') | ForEach-Object {
  if ($TestModule) {
    $Jobs += @{
      job_name = "Test PowerShell Module - $_"
      os = $_
      command = "psake test-powershell"
    }
  }

  if ($TestDocs) {
    $Jobs += @{
      job_name = "Test Documentation - $_"
      os = $_
      command = "psake test-documentation"
    }
  }
}

if ($Raw) {
  Write-Host ($Jobs | ConvertTo-JSON)
} else {
  # Output the result for consumption by GitHub Actions
  Write-Host "::set-output name=matrix::$($Jobs | ConvertTo-JSON -Compress))"
}
```


``` powershell
  [String]$FromRef
)
$Jobs = @()

$TestModule = $false
$TestDocs = $false
```

We add a new parameter called `FromRef` which specifies where in the git history we compare from. Typically this is the branch the Pull Request is targeted against. We also add two flag variables `TestModule` and `TestDocs` which we'll use to track whether we should test the Module and Documentation.

```powershell

if (![String]::IsNullOrWhiteSpace($FromRef)) {
  (& git diff --name-only $FromRef...HEAD) | ForEach-Object {
    if ($_ -like 'src/*') { $TestModule = $true }
    if ($_ -like 'docs/*') { $TestDocs = $true }
  }
}
```

If the `FromRef` is set then we run the `git diff` command.

* `--name-only` means it only returns the filename instead of the full diff for each file
* `$FromRef...HEAD` means to compare from `$FromRef`, to the current commit (This is known as [HEAD](https://git-scm.com/book/en/v2/Git-Internals-Git-References))

Then for each file that has been changed we test if it's a PowerShell Module file (`src/*`) or a documentation file (`docs/*`) and set the appropriate flag (TestModule or TestDocs)

```powershell
# Make sure we test something
if (!$TestModule -and !$TestDocs) {
  $TestModule = $true
  $TestDocs = $true
}
```

It is possible that a Pull Request doesn't change either the Module or Documentation so test everything just in case.

``` powershell
  if ($TestModule) {
    $Jobs += @{
      job_name = "Test PowerShell Module - $_"
      os = $_
      command = "psake test-powershell"
    }
  }

  if ($TestDocs) {
    $Jobs += @{
      job_name = "Test Documentation - $_"
      os = $_
      command = "psake test-documentation"
    }
  }
```

And now only add the PowerShell Module and Documentation testing if the appropriate flag is set.

The last change is to the GA Workflow.

Previously we called the PowerShell script using

``` yaml
        run: "& .github/workflows/create-test-matrix.ps1"
```

and instead now we can pass through the `-FromRef` argument

{% raw %}
``` yaml
        run: "& .github/workflows/create-test-matrix.ps1 -FromRef '${{ github.base_ref }}'"
```
{% endraw %}

The `github.base_ref` variable comes the GitHub Actions context syntax

> The base_ref or target branch of the pull request in a workflow run. This property is only available when the event that triggers a workflow run is a pull_request.

[Context Documentation](https://docs.github.com/en/free-pro-team@latest/actions/reference/context-and-expression-syntax-for-github-actions#github-context)

# Wrapping Up

I migrated from Travis and AppVeyor CI using a custom PowerShell matrix creation script which now gives me more power to cater for different testing scenarios. And we also saw what else you could achieve with this technique; using it to change testing based on who made the change, or changing the testing requirements based on what was changed.
