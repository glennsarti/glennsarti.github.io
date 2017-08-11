---
title: Update - Puppet Language Server
excerpt: "An update on my next big project: Puppet Language Server Extension"
category:
  - blog
header:
  overlay_image: /images/header-vscode.png
  overlay_filter: 0.75
  teaser: /images/teaser-vscode-update.png
tags:
  - vscode
  - puppet
  - language
  - server
  - extension
modified: 2016-08-11
---

{% include toc icon="tags" %}

It's been a while since my last blog post and I've been busy!  I have been working with [James Pogran](https://github.com/jpogran) to get a working Puppet Language Server extension for VS Code, which was part of my [Next Big Project](../next-big-project) post.

## So what have we done?

James had already written a client only Puppet extension which included all of the basic tasks: Syntax Highlighting, Snippets, Commands etc. and it was fantastic.  I talked with James and we joined forces.  At Puppet we had a Hack Day coming up and we decided to work on the Puppet Language Server together.

I worked on the Language Server and boy did we deliver!  We recently presented our work at one of our regular demo-days, and the extension was received with a lot of minds blown and applause!

I originally had the following goals and I've updated each with what we has been achieved (as of this post):

## Goals

* Get some basic [intellisense](https://code.visualstudio.com/docs/editor/editingevolved#_intellisense) for Puppet files (.pp)

  This has started.  The server has the ability to serve auto-complete and hover requests, but not all text is enabled e.g. the `class` text isn't supported by the Hover provider.
  {: .notice--warning}

  * Resource parameter/property names

  * Function names

  * Facter names (Legacy and Structured) and include their current values as [Parameter Hints](https://code.visualstudio.com/docs/editor/editingevolved#_parameter-hints)

  Done
  {: .notice--success}


* Syntax highlighting

James had already completed this
{: .notice--success}


* Run Puppet commands from the Command Palette (puppet module, puppet facts)

Not started
{: .notice--primary}


* The Client and Server will communicate over TCP, but for the moment only localhost and only for Windows platforms

Done
{: .notice--success}


## Stretch goals

* Linting

Done
{: .notice--success}


* Use VS Code on Windows to communicate with a Language Server on a Linux based OS (and vice-versa)

Done
{: .notice--success}


* [Show Errors and Warnings](https://code.visualstudio.com/docs/editor/editingevolved#_errors-warnings)

James had already completed this and the language server linting automatically adds these
{: .notice--success}


* [Code Folding](https://code.visualstudio.com/docs/editor/editingevolved#_folding)

James had already completed this
{: .notice--success}


## Source Code

The Puppet Language Server source code is in James' github account

[https://github.com/jpogran/puppet-vscode](https://github.com/jpogran/puppet-vscode)

At the time of writing this is currently in the `language-server-feature` branch.  Once we complete the initial work, we'll merge it into master.


## See the extension in use

I created several animated GIF files with showing the various parts of the extension

### Starting the extension from an empty file

![Empty File]({{ site.url }}/blog-images/puplangsrv_blank_file.gif)

Once you save the file with a `.pp` extension the language server will connect which you can see from the syntax highlighting and the Puppet version in the bottom right hand corner (`4.10.1`)


### Auto-completion of resources, parameters and parameters

![Auto-complete support]({{ site.url }}/blog-images/puplangsrv_auto_complete.gif)

* Pressing `Ctrl+Space` will start auto-complete

* `$facts` is supported and shows facts hints


### Hover information during editing

![Hover support]({{ site.url }}/blog-images/puplangsrv_hover.gif)

* Move the mouse over text to get hover information, for example: Resource, Parameter, Property, Function and Fact variables


### Puppet Linting as you type

![Live Puppet Lint]({{ site.url }}/blog-images/puplangsrv_puppet_lint.gif)

* Puppet Lint errors and warnings are evaluated as you type instead of only when you save the document.  Note how the green squiggly line (lint warning) disappears once the hash rocket (`=>`) is moved into the correct place.


### Support for `puppet resource`

![Puppet Resource]({{ site.url }}/blog-images/puplangsrv_puppet_resource.gif)

* Just like the original extension, you can run `Puppet: Resource` to automatically insert puppet resource statements.


### Puppet node graph preview

![Puppet Node Graph Preview]({{ site.url }}/blog-images/puplangsrv_node_graph.gif)

* Puppet Enterprise introduced a feature which showed the graph that Puppet used when applying a catalog.  Now you can preview the node graph for a manifest, while you edit it!!  Open the command palette with `Ctrl-Shift-P` and type `puppet node` and open the preview to the side.


### Cross Platform Support for the language server

![Cross Platform Support]({{ site.url }}/blog-images/puplangsrv_cross_plat.gif)

* You can change the `puppet.languageserver.address` to change which server the client will connect to.  In this example the client connects to a server at `192.168.200.52` which was a local Ubuntu VM.


## What's next?

We have enough features in the extension so James and I are just concentrating on stabilising the language client and server so we can release this on the VS Code Marketplace as soon as possible.
