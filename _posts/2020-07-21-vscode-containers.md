---
title: New feature for VSCode Remote Containers
excerpt: Opening VSCode Projects as a volume in VSCode
category:
  - blog
header:
  image: /images/header-vscode.png
  overlay_image: /images/header-vscode.png
  teaser: /images/teaser-vscode-update.png
tags:
  - "2020"
  - wsl2
  - vscode
---

## I was about to ...

So I was about to write a post about a book I had read recently. I fired up Docker Desktop and VSCode to start writing and then I saw a new option:

![Clone in Volume screenshot]({{ site.url }}/blog-images/vscode-clone-in-volume.png){: .align-center}

After much digging (Far more than I should've to be honest!) I found this small sentence in the [VSCode 1.47 release notes](https://code.visualstudio.com/updates/v1_47#_remote-development)

> Remote - Containers: Prompt to open repository in a volume.

After more digging (Really? It shouldn't be this hard to find this stuff...) I found the more [detailed release notes](https://github.com/microsoft/vscode-docs/blob/master/remote-release-notes/v1_47.md#containers-version-01280)

> *Guidance to open a repository in a volume*
>
> When opening a Git repository folder, the Reopen in Container notification now offers to clone and reopen the repository in a Docker volume. Using a Docker volume has better disk performance because it uses a Linux filesystem without any extra layer between the Linux Kernel and the filesystem. (We do not show this guidance on Linux, but the feature is still available using the Remote-Containers: Open Repository in Container command.)

## So what?

Now some people may be thinking "So what?", but this is a Big Deal™ . I had recently stopped using WSL2 as the Docker Backend as it had a [major drawback with file system watchers](https://github.com/microsoft/WSL/issues/4169): The first being VSCode's own file watcher so it can automatically refresh git status, and the second being for Jekyll to automatically rebuild my blog while I wrote.

While I put in a workaround for Jekyll (Force to use polling instead of a watcher) there was no way I could get around the VSCode Git problem.

<blockquote class="twitter-tweet"><p lang="en" dir="ltr">Hrmm, I&#39;m pretty sure I&#39;m hitting a variant of <a href="https://t.co/I5hK87jYqT">https://t.co/I5hK87jYqT</a><br><br>File Watchers in Windows File space don&#39;t appear to be working...</p>&mdash; Glenn Sarti (@GlennSarti) <a href="https://twitter.com/GlennSarti/status/1276786181334683648?ref_src=twsrc%5Etfw">June 27, 2020</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

---

So what does this new feature do? Well previously the Remote Container extension would mount my Windows project directory into a Linux Container, which meant inside the container it was using the 9p filesystem which has the issues ...

But now, it looks like VSCode will clone a copy of my project INTO the container, which means it's on the "native" container filesystem so my issues have all gone! Sure, I'm still subject to WSL2 performance issues, but it's so much better than having a full Hyper-V VM. This also means it's a _copy_ of my local project, so even though it _looks_ like my local project, you can't just add and remove files like you could previously.

## Thankyou 🙏🙏

So thanks VSCode Team! This is a great improvement for me!
