---
title: Configuring WSL for ruby
excerpt: Quickly setting up WSL for ruby development
category:
  - blog
header:
  image: /images/header-wsl.png
  overlay_image: /images/header-wsl.png
  overlay_filter: 0.5
  teaser: /images/teaser-wsl-and-ruby.png
tags:
  - ubuntu
  - ruby
  - windows
  - puppet
---

I use an Apple Mac laptop for software development, and sometimes it can be awkward switching between OSX and Windows Virtual Machines.  Sure, there are many tools available which help transitioning. However I miss the native Windows experience, but I still need access to some linux based ruby tools.

### Enter Windows Subsystem for Linux

Windows 10 has introduced a new feature for Insiders called [Windows System for Linux](https://msdn.microsoft.com/en-us/commandline/wsl/faq), WSL for short.  While still in Beta, this allows Windows to run _native_ ubuntu binaries on Windows, that is, running Linux as a subsystem on Windows.  To be fair this isn't exactly a new feature - Windows NT had a POSIX subsystem, and later changed to Windows Services for Unix.

WSL is best described as what it is not:

* WSL is not a Virtual Machine running in the background, like [MED-V](https://technet.microsoft.com/en-us/windows/hh826073.aspx)

* WSL is not Unix utilities recompiled for Windows, like [Cygwin](https://www.cygwin.com/)

* WSL is not a Thunking or Shimming layer, like [Windows On Windows](https://en.wikipedia.org/wiki/Windows_on_Windows), or WoW64 which allows 32bit apps to run on a 64bit operating system

* WSL is not transpiling [ELF](https://en.wikipedia.org/wiki/Executable_and_Linkable_Format) for use on Windows


### How do I install WSL?

Follow the instructions on the [installation guide](https://msdn.microsoft.com/en-us/commandline/wsl/install_guide)

Prerequisites at the time of writing this blog;

1. Your PC must have an AMD/Intel x64 compatible CPU

2. You must be a member of the (free) Windows Insider Program (Preferably Fast-Ring)

3. Your PC must be running a 64-bit version of Windows 10 Anniversary Update build 14316 or later


## Installing various versions of Ruby on WSL

Installing Ruby of WSL is mostly the same as if you were installing Ruby on a regular Ubuntu based computer.

These instructions were adapted from [Hervé Leclerc](https://medium.com/@hleclerc/use-rbenv-ruby-on-windows-10-linux-wsl-a9bce8d97300#.gf9qu9bzu)

The ruby installation can take **a long time** and at times sits at 100% CPU. You can `tail -f` the installation log file in `/tmp/ruby-build.xx.log` to make sure the installation is progressing and if any errors were raised.
{: .notice--warning}

From a WSL bash prompt run the following commands;

``` bash
# Assumes a clean WSL image

# Install all of the development dependences.  Mainly required for compiling ruby gems with native extensions
sudo apt-get install -y build-essential git libreadline-dev
sudo apt-get install -y libssl-dev zlib1g-dev autoconf libicu-dev
sudo apt-get install -y git-core curl zlib1g-dev build-essential libssl-dev libreadline-dev libyaml-dev libsqlite3-dev sqlite3
sudo apt-get install -y libxml2-dev libxslt1-dev libcurl4-openssl-dev python-software-properties libffi-dev

# Install rbenv and a few helpful plugins
# This will enable switching between different ruby versions
git clone https://github.com/rbenv/rbenv.git ~/.rbenv
git clone https://github.com/rbenv/ruby-build.git ~/.rbenv/plugins/ruby-build
git clone https://github.com/rkh/rbenv-whatis.git ~/.rbenv/plugins/rbenv-whatis
git clone https://github.com/rkh/rbenv-use.git ~/.rbenv/plugins/rbenv-use
cd ~/.rbenv && src/configure && make -C src
echo 'export PATH=$HOME/.rbenv/bin:$PATH"' >> ~/.bash_profile
eval "$(rbenv init -)"

# Let's install some Ruby's - 1.9.3 for backward compatibility and 2.3.1 as the latest
rbenv install 2.3.1
rbenv install 1.9.3-p551
# rbenv requires a rehash after installing a ruby version
rbenv rehash

# Go through each ruby version and install bundler globally as we'll need it later
rbenv global 1.9.3-p551
rbenv use 1.9.3-p551
gem install bundler

rbenv global 2.3.1
rbenv use 2.3.1
gem install bundler
```

If this a fresh git installation don't forget to configure your git settings such as email and name.
{: .notice--info}

### Additional packages for Puppet development

If you plan on doing some [Puppet](https://www.puppet.com) you'll need some extra packages:

``` bash
# Puppet:
sudo apt-get install -y autoconf

# DSC Module:
sudo apt-get install -y libicu-dev
```

### Upgrading the git client

The version of Git on WSL is a little old (1.9.2) but you can upgrade it:

``` bash
sudo add-apt-repository ppa:git-core/ppa -y
sudo apt-get update
sudo apt-get install -y git
```

This upgrades git to the latest version on Ubuntu.

### Using ruby on WSL

To use Ruby on WSL is exactly the same as you would use Ruby on a regular Unbuntu computer.

Here's an example of a bundle install:

* In Windows, clone the git [Puppet repository](https://www.github.com/puppetlabs/puppet) to `C:\Source\puppet`

* Open a WSL bash prompt and change directory to the `puppet` repository.  Remember the `C:\` in Windows is mounted in `/mnt/c` in WSL, and that paths are case sensitive in Ubuntu.

``` bash
glennsarti@GLENNSARTI:~$ cd /mnt/c/Source/puppet
glennsarti@GLENNSARTI:/mnt/c/Source/puppet$
```

* Set the ruby version to 2.3.1 with

``` bash
glennsarti@GLENNSARTI:/mnt/c/Source/puppet$ rbenv global 2.3.1
glennsarti@GLENNSARTI:/mnt/c/Source/puppet$ rbenv version
2.3.1 (set by /home/glennsarti/.rbenv/version)
glennsarti@GLENNSARTI:/mnt/c/Source/puppet$
```

* Run a ruby bundle install

``` bash
glennsarti@GLENNSARTI:/mnt/c/Source/puppet$ bundle install --path .bundle/gems
Fetching gem metadata from https://rubygems.org/
Fetching version metadata from https://rubygems.org/
Resolving dependencies...
Installing rake 10.1.1
Installing CFPropertyList 2.2.8
Installing addressable 2.4.0
Installing ast 2.3.0
Installing builder 3.2.2
Installing coderay 1.1.1
Installing safe_yaml 1.0.4
Installing diff-lcs 1.2.5
Installing facter 2.4.6
Installing hashdiff 0.3.0
Installing json_pure 2.0.2
Using json 1.8.3
Installing json-schema 2.1.1
Installing metaclass 0.0.4
Installing method_source 0.8.2
Installing msgpack 1.0.0 with native extensions
Installing multi_json 1.7.7
Installing net-ssh 2.9.4
Installing powerpack 0.1.1
Installing slop 3.6.0
Installing puppet-lint 2.0.0
Installing rspec-support 3.5.0
Installing racc 1.4.9 with native extensions
Installing rack 1.6.4
Installing rainbow 2.1.0
Installing redcarpet 2.3.0 with native extensions
Installing ruby-progressbar 1.8.1
Installing unicode-display_width 1.1.0
Installing ruby-prof 0.15.9 with native extensions
Installing thread_safe 0.3.5
Installing vcr 2.9.3
Installing yard 0.9.5
Using bundler 1.12.5
Installing puppet-syntax 2.1.0
Installing parser 2.3.1.2
Installing crack 0.4.3
Installing hiera 3.2.0
Installing rdoc 4.2.2
Installing mocha 0.10.5
Installing pry 0.10.4
Installing rspec-core 3.5.2
Installing rspec-expectations 3.5.0
Installing rspec-mocks 3.5.0
Installing tzinfo 1.2.2
Installing rubocop 0.39.0
Installing webmock 1.24.6
Using puppet 4.5.4 from source at `/mnt/c/Source/puppet`
Installing rspec-collection_matchers 1.1.2
Installing rspec-its 1.2.0
Installing rspec 3.5.0
Installing rspec-puppet 2.4.0
Installing rspec-legacy_formatters 1.0.1
Installing yarjuf 2.0.0
Installing puppetlabs_spec_helper 1.1.1
Bundle complete! 34 Gemfile dependencies, 54 gems now installed.
Bundled gems are installed into ./.bundle/gems.
Post-install message from yard:
--------------------------------------------------------------------------------
As of YARD v0.9.2:

RubyGems "--document=yri,yard" hooks are now supported. You can auto-configure
YARD to automatically build the yri index for installed gems by typing:

    $ yard config --gem-install-yri

See `yard config --help` for more information on RubyGems install hooks.

You can also add the following to your .gemspec to have YARD document your gem
on install:

    spec.metadata["yard.run"] = "yri" # use "yard" to build full HTML docs.

--------------------------------------------------------------------------------
Post-install message from rdoc:
Depending on your version of ruby, you may need to install ruby rdoc/ri data:

<= 1.8.6 : unsupported
 = 1.8.7 : gem install rdoc-data; rdoc-data --install
 = 1.9.1 : gem install rdoc-data; rdoc-data --install
>= 1.9.2 : nothing to do! Yay!
glennsarti@GLENNSARTI:/mnt/c/Source/puppet$
```

Notice the native extensions being compiled and installed

* Run a simple bundle exec

``` bash
glennsarti@GLENNSARTI:/mnt/c/Source/puppet$ bundle exec puppet --version
4.5.4
glennsarti@GLENNSARTI:/mnt/c/Source/puppet$
```

## Possible problems and resolutions

### Error: Parent directory is world writable but not sticky

On your first `bundle install` you may receive the error message:

```
ArgumentError: parent directory is world writable but not sticky
```

This is a known error on WSL.  The fix mentioned in [Bundler Github Issue 4599](https://github.com/bundler/bundler/issues/4599) resolves the issue

```bash
find ~/.bundle/cache -type d -exec chmod 0755 {} +
```


### WSL appears a bit slow, particularly accessing files or during bundle installs

WSL generates a lot of Disk IO, particularly during a bundle install with native extensions.  Unfortunately Windows Defender is actively scanning those files for malware etc..  I would recommend adding an exception to Windows Defender for the WSL files and folders as I don't feel Defender will add much value.  My reasoning is: I wouldn't expect Defender to detect an Ubuntu based virus in a linux binary on Windows, therefore there's not value in scanning it at all.

Add a [Windows Defender folder exemption](https://support.microsoft.com/en-us/instantanswers/64495205-6ddb-4da1-8534-1aeaf64c0af8/add-an-exclusion-to-windows-defender) for `%LOCALAPPDATA%\lxss`.  In my case it was at `C:\Users\GlennSarti\AppData\Local\lxss`
