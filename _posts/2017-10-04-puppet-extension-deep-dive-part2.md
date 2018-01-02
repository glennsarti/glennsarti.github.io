---
title: VS Code Puppet Extension Deep Dive - Part 2
excerpt: Introduction to the Language Server
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
modified: 2018-01-02
---

{% include custom/mermaid.html %}
{% include toc icon="tags" %}

# Introduction to the Language Server

This is part 2 of a series about the Puppet Extension for Visual Studio Code.  See the bottom of this post for links to all other posts.

[Source Code](https://github.com/jpogran/puppet-vscode)

[Extension on the VS Code Marketplace](https://marketplace.visualstudio.com/items?itemName=jpogran.puppet-vscode)

Initially the Puppet extension while useful lacked some features, namely;

* Checking for code problems while typing as opposed to only on document save

* Could only use static code snippets

* No or very limited auto-complete

* No hover support

Fortunately Puppet has most of these abilities available within its codebase, in particular, an entire lexer and parser for the Puppet language.  We can then augment Puppet with [puppet-lint](http://puppet-lint.com/) to provider syntax and code linting in real-time.  However Puppet is written in Ruby, while VS Code extensions are written in TypeScript.  This is where a Language Server comes into play.

## What is a language server

The VS Code documentation gives a very concise definition of a language server and when they should be used;

> Language servers allow you to add your own validation logic to files open in VS Code. Typically you just validate programming languages. However validating other file types is useful as well. A language server could, for example, check files for inappropriate language.
>
> In general, validating a programming language can be expensive. Especially when validation requires parsing multiple files and building up abstract syntax trees. To avoid that performance cost, language servers in VS Code are executed in a separate process. This architecture also makes it possible that language servers can be written in other languages besides TypeScript/JavaScript and that they can support expensive additional language features like code completion or Find All References.

[Source](https://code.visualstudio.com/docs/extensions/example-language-server)

So, in order to get the more advanced features for the extension we needed to implement a Language Client in the extension and a Language Server which the client would communicate with.

<div class="mermaid">
graph LR
  subgraph VS Code Extension
    lc(Language Client)
  end

  lc-->ls(Language Server)

  style lc fill:#ccf,stroke:#333
  style ls fill:#fcc,stroke:#333
</div>

Fortunately, there are many other language servers available (PowerShell, Go, PHP, C#) which means we were able to use these as inspiration for how to write our Puppet language server.

## Overview of the Puppet Language Server

The Puppet language server complies with version 3.0 of the [Language Server Protocol](https://github.com/Microsoft/language-server-protocol/blob/master/protocol.md) although not all capabilities are implemented; for example, the `documentSymbol` provider is not available to the client.

The architecture of the puppet language server is that:

* The de facto communication method used is pipes (Stdin/Stdout) however ruby can have issues with disparate text encoding on Windows operating systems.  Instead we settled on using sockets (TCPIP) as the transport method.  This meant we could safely use strings with different encodings, leverage other language servers (The PowerShell language server uses TCPIP) and potentially we could run the language server on a different host.  This means a user could be writing in VS Code on a Windows computer, but communicating with a Linux computer over the network.

* It _only_ requires the ruby environment from the Puppet Agent to be able to run.  This means gems with Native Extensions can not be used, and that any gems not explicitly supplied by Puppet Agent must be vendored inside the language server.  Note that we are currently investigating the use of the Puppet SDK instead; however at the time of first writing the server, the PDK was not available.

* The Language Server is written in a similar style to the [OSI Network Model](https://en.wikipedia.org/wiki/OSI_model) where the lower layers implement a transport (TCPIP) and a frame (JSON-RPC), while the top layers respond to requests and integrate with Puppet

<div class="mermaid">
graph TB

subgraph VS Code Extension
  lc(Language Client)
end

subgraph Puppet Language Server
  layer0(TCP Server)
  layer1(Generic client connection)
  layer2(JSON RPC Handler)
  layer3(Message Router)
  layer4(Providers)
  layer5(Puppet Helpers)
  layer6(Puppet / Facter / Hiera)

  layer0-->layer1
  layer1-->layer2
  layer2-->layer3
  layer3-->layer4
  layer4-->layer5
  layer5-->layer6
end

lc-->layer0

style lc fill:#ccf,stroke:#333
style layer0 fill:#fcc,stroke:#333
style layer1 fill:#fcc,stroke:#333
style layer2 fill:#fcc,stroke:#333
style layer3 fill:#fcc,stroke:#333
style layer4 fill:#fcc,stroke:#333
style layer5 fill:#fcc,stroke:#333
style layer6 fill:#fcc,stroke:#333
</div>

### TCP Server

The TCP server (`simple_tcp_server.rb`) is responsible for listening on a TCP port and then transferring raw data to/from client socket to the `Generic client connection` layer

### Generic client connection

The client connection (`simple_tcp_server.rb`) exposes very simple methods to receive data from a client, send data to a client and close a connection to a client.

### JSON RPC Handler

The JSON RPC Handler (`json_rpc_handler.rb`) receives the raw bytes from the connection and extracts the [JSON RPC](http://www.jsonrpc.org/specification) headers and message and passes it to the `Message Router` layer.  Conversely, the handler will take response from the `Message Router` and wrap them in the required JSON RPC headers prior to transmission.

### Message Router

The message router (`message_router.rb`) receives requests and notifications, as defined by the [language server protocol](https://github.com/Microsoft/language-server-protocol/blob/master/protocol.md#base-protocol-json-structures) and then either deals with the messages directly or calls on a `Provider` to calculate the correct response.

The `Message Router` also holds a virtual document store, which contains an in-memory copy of the documents being edited on the language client.

### Providers

The provider classes (`completion_provider.rb`, `document_validator.rb`, `hover_provider.rb`) implement the various puppet languages services.  Typically they mirror the language service services, e.g. the `textDocument/hover` request is serviced by the hover provider.

### Puppet Helpers

Many of the providers use common code.  The Puppet Helpers (`facter_helper.rb`, `puppet_helper.rb`, `puppet_parser_helper.rb`) abstract away common tasks which make the providers smaller and easier to edit.  The helpers also provider a caching layer so that
Puppet and Facter do not need to be evaluated as often, for example getting all the facter facts.

### Puppet / Facter / Hiera

At this layer any calls are using Puppet directly, for example; Calling facter or executing a catalog compilation.

# The TCP Server

When I initially started writing a Language Server, I used the [Event-Machine gem](https://rubygems.org/gems/eventmachine).  It was very easy to use and worked on all platforms.  However, Event Machine used native gems which meant, while I could use it during development, it would _not_ work when using the Puppet ruby environment.  I did try the pure ruby Event Machine implementation but there were bugs when running on Windows which could not be easily fixed.

So, I had to look for pure ruby socket server.  Unfortunately, there were no gems which provided this feature, they all used native gems.  Eventually I came across a [Stack Overflow article](https://stackoverflow.com/questions/29858113/unable-to-make-socket-accept-non-blocking-ruby-2-2) (A developers favourite resource!). User [Mystic](https://stackoverflow.com/users/4025095/myst) posted a working pure ruby socket server, which I adapted for a Language Server.

## Implementation

The TCP Server is basically a simple synchronous socket server which invokes other methods, on new threads, as socket events occur.  Socket events would be; A client connects, a client sends data, a client disconnects and an error occurred.  The TCP Server is only concerned with the most basic of TCP functions, so all interpretation of the data happens it later layers (Generic Client connection and JSON RPC handler)

The `PuppetLanguageServer::SimpleTCPServer` class is the main entry point and only has a few public methods.

### Public methods

*`initialize`*

The initialize method creates an instance of the server

*`add_service`*

``` ruby
add_service(hostname = '127.0.0.1', port = 8081, parameters = {})
```

The add_service method instructs the TCP Server to listen on the interface and port number specified.  To listen on all interfaces specify `0.0.0.0`.  In the Language Server `parameters` is not used

*`start`*

``` ruby
start(handler = PuppetLanguageServer::SimpleTCPServerConnection, connection_options = {}, max_threads = 2)
```

The start method is a blocking call which;

* Creates the required threads to deal with socket events
* Preserves the connection options, which are passed along later when socket events occur
* Creates an instance of the handler class on each socket event.  This is important to remember as you can not save state within the handler itself because, potentially, the object is a new instance and did not preserve the state

*`stop_services`*

The stop_services does pretty much what it says.  It stops listening on all services and unblocks the `start` method

### Usage

In the Puppet Language server we create the TCP server using the following code;

``` ruby
def self.rpc_server(options)
  log_message(:info, 'Starting RPC Server...')

  server = PuppetLanguageServer::SimpleTCPServer.new

  server.add_service(options[:ipaddress], options[:port])
  trap('INT') { server.stop }
  server.start(PuppetLanguageServer::MessageRouter, options, 2)

  log_message(:info, 'Language Server exited.')
end
```

The trap statement calls `server.stop` which is a bug as it should be `server.stop_services`.  I'll get this fixed up
{: .notice--danger}

So, we are creating;

* A TCP Service which listen on host `options[:ipaddress]` on port `options[:port]`
* The handler class is `PuppetLanguageServer::MessageRouter` which implements layers 2, 3 and 4 in our diagram (Later blog posts will delve into the message router)
* Create 2 threads for handling client connections.  As this is a Language Server not a full featured web server there should only be one client connecting so the request rate will be relatively small

## Additional features of the TCP Server

The TCP Server also includes some new features I added to the original source;

* The Server will optionally exit when the last client disconnects.  This is specified in the `options[:stop_on_client_exit]` property
* The Server will optionally exit if no clients initially connect within a specified timeout.  This is specified in the `options[:connection_timeout]` property
* Unified logging using `PuppetLanguageServer.log_message` method
* Setting the socket information for call handlers so that they can send data back to the client easily
* Setting the parent server information in the call handler so that clients can request the server to drop it's connection

# The Generic client connection

The handler class specified when starting the TCP server should inherit from the abstract `SimpleTCPServerConnection` class.  This class exposes some basic methods and properties which later layers can use.  There isn't too much to describe here and the method names, hopefully, speak from themselves;

* `post_init` - Raised after a client connections
* `receive_data(data)` - Raised when a client sends data to the server
* `send_data(data)` - Invoked by the server to send data to a client
* `error?` - Returns a boolean indicating if the socket is in an error state
* `close_connection_after_writing` - Invoked by the server to send remaining data to the client and then close the connection
* `close_connection` - Invoked by the server to terminate the client connection immediately
* `unbind` - Raised when a client disconnect

While these parameters are accessible, they are typically not needed by later layers.

* `socket` - The raw socket object for this connection
* `simple_tcp_server` - The owner TCP Server object for this connection

Ruby doesn't really have abstract classes so by default the `SimpleTCPServerConnection` class will simply log incoming data and send outgoing data.

# Wrapping up...

That wraps it up for the beginning of the Language Server.  In this post I introduced the architecture for the server and described the first two layers which provided basic TCP services for the server.  In the next post we'll look at the JSON RPC handler and message router

# Blog series links

[Part 1 - Introduction to the extension](../puppet-extension-deep-dive-part1)

[Part 2 - Introduction to the Language Server](../puppet-extension-deep-dive-part2)

[Part 3 - JSON RPC handler and message router](../puppet-extension-deep-dive-part3)

Part 4 - Language Providers
