---
title: VS Code Puppet Extension Deep Dive - Part 4
excerpt: WE'VE MOVED!
category:
  - blog
header:
  overlay_image: /images/header-vscode.png
  overlay_filter: 0.75
  teaser: /images/teaser-vscode-puppet.png
tags:
  - vscode
  - puppet
  - language
  - server
  - extension
---

# Welcome to Lingua Pupuli

![Lingua-Pupuli Logo](https://avatars3.githubusercontent.com/u/37545262?s=100&v=4){: .align-left}

It's been a while since the last update and part of the delay was; the VS Code Extension has moved.  Previously all of the code lived in [https://github.com/jpogran/puppet-vscode](https://github.com/jpogran/puppet-vscode); the extension, the language server, debug server etc..  What we've done is move to a new GitHub organisation - [Lingua Pupuli](https://github.com/lingua-pupuli) - The Language of the Puppet.  We also split out the code into three separate projects;

* [puppet-vscode](https://github.com/lingua-pupuli/puppet-vscode)

This project is the Puppet VS Code extension.  It contains all of the code for the extension itself.  Shared resources, such as the Editor Services and Syntax Files are vendored in.

* [puppet-editor-services](https://github.com/lingua-pupuli/puppet-editor-services)

This project contains the ruby code to run a Language Server and Debug Server for text editors.  This project is designed to be consumed by editor extensions or plugins that can then surface the features into their respective text editors.

* [puppet-editor-syntax](https://github.com/lingua-pupuli/puppet-editor-syntax)

This project contains the Textmate Syntax Highlighting files for Puppet Language in various formats.  The master format is the XML PList style, which is used by VS Code, but it also has YAML, JSON, CSON and Atom flavoured CSON.  This project is designed to be consumed by editor extensions or plugins

# What does this mean for the blog series?

It means I can now focus on writing the blog posts instead of updating the code (Documentation is a good thing!).  The blog series will now reference the [puppet-editor-services](https://github.com/lingua-pupuli/puppet-editor-services) project. I've gone back through the previous posts and updated them with the new repository locations but as part of the move I took the opportunity to add some release features and move files around to make more sense.

## Release Process

Now that the Editor Services can be released independant of the VSCode Extension we needed a release process.  We took the following steps;

### Continue the version numbering

It didn't make sense to reset the version back to 0.0.1, or something else.  This meant we could back through the commit history of the new project and retrospectively tag the project at the relevant commit points, so the code only diverges after version 0.10.0.

``` text

Puppet Editor Services

v0.8.0        v0.9.0       v0.10.0     v0.11.0     v0.12.0
   |-------------|------------|-----------|-----------|------->

   |-------------|------------|---------------------------|--->
v0.8.0        v0.9.0       v0.10.0                     v0.11.0

Puppet VSCode Extension
```

Note how the versioning diverges after v0.10.0
{: .notice}

### Add release assets

Previously the Language Server etc. was only available through the source code.  Now it is available as a tarball or ZIP file via the GitHub [releases page](https://github.com/lingua-pupuli/puppet-editor-services/releases).

[Link](https://github.com/lingua-pupuli/puppet-editor-services/pull/7)

## Moving around source files

### Moving code to PuppetEditorServices namespace

Previously the shared code between the Language Server and Debug Server were stored in the ruby namespace `PuppetVSCode`.  As we've now split out of the VS Code repository this didn't make sense, so it was renamed to `PuppetEditorServices`.

[Link](https://github.com/lingua-pupuli/puppet-editor-services/pull/1)

### Moving provider and helper code to per-document type namespace

Previously all of the providers (hover, autocomplete etc.) were mixed in the root directory of the language server.  This made it difficult to manage and understand where to edit pieces of code.  What we did is move them to a directory which represents what kind of document they process, for example, Puppet Manifests, EPP ([Embedded Puppet Template](https://puppet.com/docs/puppet/5.3/lang_template_epp.html)) or [Puppetfile](https://github.com/puppetlabs/r10k/blob/master/doc/puppetfile.mkd).  So the new layout looks like;

``` text
lib
  +- puppet-languageserver
       +- epp
       |    < Contains providers for the EPP template language >
       |
       +- manifest
       |    < Contains providers for the Puppet manifest language >
       |
       +- puppetfile
            < Contains providers for the Puppetfile language >
```

[Link](https://github.com/lingua-pupuli/puppet-editor-services/pull/23)

# Blog series links

[Part 1 - Introduction to the extension](../puppet-extension-deep-dive-part1)

[Part 2 - Introduction to the Language Server](../puppet-extension-deep-dive-part2)

[Part 3 - JSON RPC handler and message router](../puppet-extension-deep-dive-part3)

[Part 4 - Welcome to Lingua Pupuli](../puppet-extension-deep-dive-part4)

Part 5 - Language Providers
