---
title: Next big project
excerpt: My next big project ...
category:
  - blog
header:
  overlay_image: /images/header-vscode.png
  overlay_filter: 0.75
  teaser: /images/teaser-vscode.png
tags:
  - vscode
  - puppet
  - extension
---

Not so much a blog post this time, more a note of things to come.  I've been thinking of a project I'd like to build.  I've done some work with containers and I stil love my Neo4j graph databases and of course Windows related things.  But I wanted something, a bit more challenging...

So I decided on writing [Visual Studio Code Language Client and Server](https://code.visualstudio.com/docs/extensions/example-language-server) for Puppet.

I'll keept the blog update with my progress...

### Why?

Writing Puppet manifests is ok, but it could really be so much easier with things like intellisense and parameter completion etc. and access to current Facter facts.  Also it would be nice if you could run Puppet commands from inside VS Code instead of running a seperate console.

* Puppet-Agent already installs Ruby and some gems so there's no overhead to install Ruby

* Ruby supports many types of pipes STDIN/STDOUT, Named Pipes and TCP based pipes so we can talk between Ruby and VS Code easy enough

* There's a lot of prior art I can use, particularly the PowerShell Language Server as I can code in C#

* I'd like to have this ready for the Puppet Contributor Summit in October 2017

* Puppet already exposes Abstract Syntax Tree (AST) and I can use the code in the Puppet REPL to help


### So what do I want to build

High level goals

* Get some basic [intellisense](https://code.visualstudio.com/docs/editor/editingevolved#_intellisense) for Puppet files (.pp)

  - Resource parameter/property names

  - Function names

  - Facter names (Legacy and Structured) and include their current values as [Parameter Hints](https://code.visualstudio.com/docs/editor/editingevolved#_parameter-hints)

* Syntax highlighting

* Run Puppet commands from the Command Palette (puppet module, puppet facts)

* The Client and Server will communicate over TCP, but for the moment only localhost and only for Windows platforms

Stretch goals

* Linting

* Use VS Code on Windows to communicate with a Language Server on a Linux based OS

* [Show Errors and Warnings](https://code.visualstudio.com/docs/editor/editingevolved#_errors-warnings)

* [Code Folding](https://code.visualstudio.com/docs/editor/editingevolved#_folding)

