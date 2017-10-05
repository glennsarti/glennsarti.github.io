---
title: VS Code Puppet Extension Deep Dive - Part 1
excerpt: Diving deep into the VS Code extension for Puppet - Part 1
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
modified: 2017-10-04
---

{% include toc icon="tags" %}

[James Pogran](https://github.com/jpogran) and I have been busy writing new features and fixing bugs in the Puppet VS Code extension.  So for the next series of blog posts I am going to dive deep into the source code for the extension and explain how parts of it work and how to setup a development environment for the extension.

But first some handy links:

[Source Code](https://github.com/jpogran/puppet-vscode)

[Extension on the VS Code Marketplace](https://marketplace.visualstudio.com/items?itemName=jpogran.puppet-vscode)

[The original Next Big Project blog post](../next-big-project)

# What is a VS Code extension?

The VS Code editor has an easy to use [extension system](https://code.visualstudio.com/docs/extensions/overview) to add features.  In our case we wanted to add Puppet language support to the editor so that we could give a more immersive experience for people writing Puppet manifests and modules.  For the initial work we concentrated on the client side Extension and Language Server;

## Extension

> All extensions when activated run in our shared extension host process. This separate process for extensions ensures that VS Code remains responsive through-out.
>
> Extensions include support for:
>
> Activation - load an extension when a specific file type is detected, when a specific file exists, or when a command is selected via the Command Palette or a key combination
> Editor - work with the editor's content - read and manipulate text, leverage selection(s)
> Workspace - access open editors, the status bar, information messages and more
> Eventing - connect to the editor life-cycle events such as: open, close, change, and more
> Evolved editing - create providers for rich language support including IntelliSense, Peek, Hover, Diagnostics and much, much more

[Source](https://code.visualstudio.com/docs/extensions/overview#_extensions)

For Puppet the extension would include:

* Syntax highlighting
* Static code snippets
* metadata.json schema validation

These editing experiences run inside the VS Code editor and do not rely on knowing how Puppet works, that is, they don't query Puppet for information.  The extension is written in TypeScript (NodeJS).  While writing TypeScript is outside the scope of this blog series, there is more specific information about writing extensions in the [Extensibility Principles and Patterns](https://code.visualstudio.com/docs/extensionAPI/patterns-and-principles) document.

## Language Server

> Language servers let you create a dedicated process for your extension. This is a useful design choice for your extension when your extension runs high cost CPU or IO intensive tasks which could slow other extensions. This is common for tasks that work across all files in a workspace e.g. linters or static analysis suites.

[Source](https://code.visualstudio.com/docs/extensions/overview#_language-servers)

For Puppet the langauge server would include:

* Auto-complete
* Linting
* Hover support
* Including facts in the editor
* Cross platform support, that is, running the language server on a different host than the VS Code editor

These editing experiences require an instance of Puppet, therefore it can not be used internally in VS Code.  Instead these are served from a Language Server, which is written in Ruby to better take advantage of Puppet.

## Debug Server

Currently being worked on 😉

# General Information

## Source code layout

The source code is located on [Github](https://github.com/jpogran/puppet-vscode) and is composed of the VS Code Extension in the [`client/`](https://github.com/jpogran/puppet-vscode/tree/master/client) directory and the Puppet Language Server in the [`server/`](https://github.com/jpogran/puppet-vscode/tree/master/server) directory

The majority of the client code is contained in the [`client/src`](https://github.com/jpogran/puppet-vscode/tree/master/client/src) directory, and the [`package.json`](https://github.com/jpogran/puppet-vscode/blob/master/client/package.json) file.  Don't worry, we'll dive into these files later.

## Existing documentation

The extension [Readme](https://github.com/jpogran/puppet-vscode/blob/master/README.md) file, the [Language Server Readme](https://github.com/jpogran/puppet-vscode/blob/master/server/README.md) and [Language Server documentation](https://github.com/jpogran/puppet-vscode/tree/master/server/docs) contain installation and development instructions for both the extension and Puppet Language Server.  Instead of repeating that here, I suggest you have a quick read of them.

# Extension Deep Dive

## Language Contribution

Puppet supplies a [language contribution](https://code.visualstudio.com/docs/extensionAPI/extension-points#_contributeslanguages) to VS Code

Firstly the `package.json` file snippet defines where the contribution is located:

``` typescript
    "contributes": {
        "languages": [
            {
                "id": "puppet",
                "aliases": [
                    "Puppet",
                    "puppet"
                ],
                "extensions": [
                    ".pp",
                    ".epp"
                ],
                "configuration": "./languages/puppet.configuration.json"
            }
        ],
```

* The `id` of `puppet` is important and will be used throughout the extension
* The first listed alias of `Puppet` is used when displaying the langauge name in the editor.  You will see this in the bottom right hand corner of VS Code
* The `extensions` section lists which file extensions will activate this language contribution
* The `configuration` item specifies the relative path to a language configuration file.  In our source code this is [`client/languages/puppet.configuration.json`](https://github.com/jpogran/puppet-vscode/tree/master/client/languages/puppet.configuration.json)

The language configuration file looks similar to the following:

``` typescript
{
  "comments": {
    // symbol used for single line comment. Remove this entry if your language does not support line comments
    "lineComment": "#",
    // symbols used for start and end a block comment. Remove this entry if your language does not support block comments
    "blockComment": "#"
  },
  // symbols used as brackets
  "brackets": [
    ["{", "}"],
    ["[", "]"],
    ["(", ")"]
  ],
  // symbols that are auto closed when typing
  "autoClosingPairs": [
...
  ],
  // symbols that that can be used to surround a selection
  "surroundingPairs": [
...
  ]
}
```

This configuration file is detailed in the [language contribution](https://code.visualstudio.com/docs/extensionAPI/extension-points#_contributeslanguages) link however the content is mostly easy to understand, for example, the `#` character is used to signify comments.

## Grammar Contribution

The [grammar contribution](https://code.visualstudio.com/docs/extensionAPI/extension-points#_contributesgrammars) is used to add syntax highlighting to the Puppet language.  The grammar specification is expressed in a [TextMate](http://manual.macromates.com/en/) file

Firstly the `package.json` file snippet defines where the contribution is located:

``` typescript
    "contributes": {
        "grammars": [
            {
                "language": "puppet",
                "scopeName": "source.puppet",
                "path": "./syntaxes/puppet.tmLanguage"
            }
        ],
```

* The `language` is `puppet`.  This is defined in the previous langauges contribution
* The `scopeName` must match the scope name in the TextMate file

``` xml
<plist version="1.0">
<dict>
  ... Lots of stuff ...
  <key>scopeName</key>
  <string>source.puppet</string>
  ...
</dict>
</plist>
```

* The `path` item specifies the relative path to a language configuration file.  In our source code this is [`client/syntaxes/puppet.tmLanguage`](https://github.com/jpogran/puppet-vscode/tree/master/client/syntaxes/puppet.tmLanguage)

It is outside the scope of this blog post to describe how to write TextMate grammar files.  I believe the grammar file we use in the extension is based on one published by [Brice Figureau](https://github.com/masterzen/puppet-textmate-bundle) although it should possibly be updated to this [fork](https://github.com/puppet-textmate-bundle/puppet-textmate-bundle)

## JSON Validation Contribution

Puppet [metadata.json](https://docs.puppet.com/puppet/5.1/modules_metadata.html) files describe information about a Puppet module.  They are generally used by the Puppet Forge and during a `puppet module install` command to install module dependencies.  VS Code can natively [validate JSON](https://code.visualstudio.com/docs/extensionAPI/extension-points#_contributesjsonvalidation) files using a [JSON Schema document](http://json-schema.org/documentation.html)

Firstly the `package.json` file snippet defines where the contribution is located:

``` typescript
    "contributes": {
        "jsonValidation": [
            {
                "fileMatch": "metadata.json",
                "url": "./src/metadata-json-schema.json"
            }
        ],
```

* Unlike other contributions there is no `language` setting.  Instead we define a file matching rule.  In this case we only care that the file is called `metadata.json`
* The `Url` setting specifies where the schema document is located.  Either as a local source file, like our extension does, or a remote request, for example, to the [JSON Schema Store website](http://schemastore.org/json).

Puppet has not published a JSON Schema document so I created one based on the rules defined in [Available `metadata.json` keys](https://docs.puppet.com/puppet/5.1/modules_metadata.html#available-metadatajson-keys).  In our source code this is [`client/src/metadata-json-schema.json`](https://github.com/jpogran/puppet-vscode/tree/master/client/src/metadata-json-schema.json).  Ideally I'd like to merge and then manage this file in the [metadata-json-lint gem](https://github.com/voxpupuli/metadata-json-lint) ([Github Issue #92](https://github.com/voxpupuli/metadata-json-lint/issues/92))

## Snippets Contribution

The extension also supplies some static [code snippets](https://code.visualstudio.com/docs/extensionAPI/extension-points#_contributessnippets) to VS Code.  The dynamic auto-complete snippets are supplied by the Language Server which will be discussed in later blog posts.

Firstly the `package.json` file snippet defines where the contribution is located:

``` typescript
    "contributes": {
        "snippets": [
            {
                "language": "puppet",
                "path": "./snippets/keywords.snippets.json"
            },
            {
                "language": "json",
                "path": "./snippets/metadata.snippets.json"
            }
        ],
```

* Each snippets section specifies which configuration file to use for which language.  In our case, we have two files, one for the `puppet` language ([./snippets/keywords.snippets.json](https://github.com/jpogran/puppet-vscode/tree/master/client/snippets/keywords.snippets.json)) and the other for the `json` language ([./snippets/metadata.snippets.json](https://github.com/jpogran/puppet-vscode/tree/master/client/snippets/metadata.snippets.json)) which we use for the `metadata.json` file.

A snippet configuration file is a JSON document which is described [here](https://code.visualstudio.com/docs/editor/userdefinedsnippets#_creating-your-own-snippets) but let's look at part of the `keywords.snippets.json`.

``` typescript
  "if": {
    "prefix": "if",
    "description": "Conditional statement",
    "body": [
      "if ${1:condition} {",
      "\t${2:# when true}",
      "}",
      "else {",
      "\t${2:# when false}",
      "}"
    ]
  },
```

New we can break this down line by line:

``` json
  "if": {
```

Each snippet starts with a snippet name, in this case it is called `if`

---

``` json
    "prefix": "if",
```

The autocomplete system in VS Code needs to know when it can invoke this snippet.  This is defined by the `prefix`.  Again in this example it is the text `if`.

---

``` json
    "description": "Conditional statement",
```

A user-friendly description can be set.  This is displayed in the user interface of the auto-complete dialog
TODO Insert screen grab here

---

``` json
    "body": [
      "if ${1:condition} {",
      "\t${2:# when true}",
      "}",
      "else {",
      "\t${3:# when false}",
      "}"
    ],
```

In the `body` we define the actual text which will be inserted.  Various text escape sequences can be used e.g. `\t` is the tab character, and each element in the array is a new line.  We also make use of the [tabstop](https://code.visualstudio.com/docs/editor/userdefinedsnippets#_tabstops) snippet feature.  This the text that looks like `${1:condition}`.  This means a user can tab through the snipppet and enter text in the appropriate place.  In this example the user can press tab to cycle through the condition, true and false sections of the if statement.

This results in the following text being inserted into the Puppet manifest (Assuming two-spaces for tabs):

``` puppet
if condition {
  # when true
}
else {
  # when false
}
```

# Wrapping up...

That wraps it up for the client side only language extension code.  In the next blog post we'll look at how the Language Server is constructed and how it interfaces with the extension itself.

# Blog series links

[Part 1 - Introduction to the extension](../puppet-extension-deep-dive-part1)

[Part 2 - Introduction to the Language Server](../puppet-extension-deep-dive-part2)

Part 3 - JSON RPC handler and message router
