---
title: "Language Servers, GoLang and WASM"
excerpt: Hosting Golang based language servers using WASM
category:
  - blog
header:
  overlay_image: /images/header-default.png
  overlay_filter: 0.5
  teaser: /images/teaser-default.png
tags:
  - lsp
  - vscode
  - golang
  - wasm
read_time: true
---

## Language Servers, Go and WASM - Part 1

### Back then ...

When I first started writing the [Puppet Language Server](https://github.com/puppetlabs/puppet-editor-services) with [James Pogran](https://github.com/jpogran) one of the more annoying things we had to deal with was getting Ruby, and our source code on the users computer.
We had to force user to install Puppet Agent first before our VSCode extension worked. This was sometimes problematic;
Some people used old versions. Some people used old Operating Systems and some people didn't have internet access!
It would've been great if we could bundle Ruby and Puppet Language Server together in some cross-platform way.

And then, while I was at a conference I was told (I think it was [Geoff Huntley](https://twitter.com/geoffreyhuntley)) about the magic of [Web Assembly](https://developer.mozilla.org/en-US/docs/WebAssembly) (known as WASM).

> WebAssembly is a new type of code that can be run in modern web browsers — it is a low-level assembly-like language with a compact binary format that runs with near-native performance and provides languages such as C/C++, C# and Rust with a compilation target so that they can run on the web. It is also designed to run alongside JavaScript, allowing both to work together.

You could run Ruby programs in the Web Browser without needing Ruby installed! 🎉🎉 ... but unfortunately back then Ruby support in WASM wasn't very good and I had to abandon the idea.

### Fast forward to now ...

But that was then, and this is now! I'm now a developer at HashiCorp so I'm doing a lot more in Go. I started writing a language server for one of our languages and quickly I found a familiar problem: "How do I get my language server installed?".
And that's when I remembered about WASM! Go even has native support for compiling to WASM
