---
title: VS Code Puppet Extension Deep Dive - Part 3
excerpt: JSON RPC handler and message router
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

{% include toc icon="tags" %}

# JSON RPC handler and message router

This is part 3 of a series about the Puppet Extension for Visual Studio Code.  See the bottom of this post for links to all other posts.

[Source Code](https://github.com/jpogran/puppet-vscode)

[Extension on the VS Code Marketplace](https://marketplace.visualstudio.com/items?itemName=jpogran.puppet-vscode)

In [part 2](../puppet-extension-deep-dive-part2) we introduced the Puppet Language Server and the first two base services; a TCP Server and generic client connection. So now we can receive and transmit a stream of bytes between the Language Client, VSCode, and the Language Server.  But a stream of bytes isn't that useful.  It needs a protocol so meaningful information can be exchanged, and this is the task of the JSON RPC Handler.  Then once messages are extracted, they can be routed to various parts of the Language Server to be processed.

## JSON RPC Handler

The byte stream from the network is formed into messages based on version 2 of the JSON RPC protocol, which is defined in [this specification document](http://www.jsonrpc.org/specification).  Microsoft have a shorter version of the specification in the [language server protocol repository](https://github.com/Microsoft/language-server-protocol/blob/master/protocol.md#base-protocol)

If you've seen HTTP requests before, this format should look familar.

It is composed of a header and content part, separated by a carriage return (`\r`) and linefeed (`\n`) character

``` text
+--------+------+---------+
| Header | \r\n | Content |
+--------+------+---------+
```

The Header is composed of key-value pairs separated by a colon (`:`), and each pair terminated by a carriage return and linefeed.  There is a mandatory `Content-Length` header which describes how many bytes the Content part is.  As we're encoding bytes into strings it is important to know what text encoding is used.  The header is always ASCII, while the content defaults to UTF8, however this can be set using the `Content-Type` header, for example to set UTF16; `Content-Type:application/vscode-jsonrpc; charset=utf-16`.

So let's say you wanted to send the text "Hello World", the entire JSON message would look like:

``` text
Content-Length:11\r\n\r\nHello World
```

There is a double `\r\n` as the first one indicates the end of the header pair, and the second indicates the end of the Header part.

Conversely, to read in a stream of bytes and convert to a JSON message;

1. Keep reading bytes until a double `\r\n` is received

2. Parse the header and determine how long the Content part is from the Content-Length header

3. Keep reading bytes until the content length is read

Unfortunately there is no error checking so you don't know if bytes were corrupted.  Generally the protocol is across Standard In/Standard Out text pipes which are not subject to corruption, or bytes being lost.  However when running over TCP, corruption is possible but unlikely due to TCP already doing CRC checks.  But TCP packets being dropped is possible and that's not surfaced in the language protocol.

The JSON RPC 2.0 specification also allows for batch operations, however VS Code language client does not, so the language server does not need to implement that part of the protocol.

### Implementation

The implementation used in the [Puppet Language Server](https://github.com/jamespogran/puppet-vscode/blob/master/server/lib/puppet-languageserver/json_rpc_handler.rb) was heavily inspired by the JSON parsing in the [PowerShell Language Server](https://github.com/PowerShell/PowerShellEditorServices).  The `receive_data` method keeps appending bytes coming from the TCP Server into a buffer.  As the buffer fills, it is examined for a valid JSON RPC Header, and then makes sure the buffer contains all of the expected content.  Once there is enough data in the buffer it extracts the entire message for further processing and resets the buffer.  There are probably more effecient methods to do this, however due to the low request rate the effeciency isn't required yet.

The `parse_data` method then takes the extracted message content and converts it from a JSON string into a Ruby object.  It then performs validation that the RPC version is correct (`2.0`) and whether the message is a [Request](https://github.com/Microsoft/language-server-protocol/blob/master/protocol.md#requestmessage) or [Notification](https://github.com/Microsoft/language-server-protocol/blob/master/protocol.md#notification-message).  Then the message is sent to the Message Router for processing.

The JSON Handler also exposes a few helper methods

#### `reply_*`

These methods will send reply messages but hide all the mundane work to craft the JSON object.  For example the `reply_error` method takes an error code and error text parameter and will send an error message back to the Language Client.

#### `send_show_message_notification`

The language server can send a JSON event to popup a dialog window on the client.  This can be handy for the Server to notify the Client of fatal errors.

#### `Request` object

When a Request message is sent, the JSON RPC Handler creates a `Request` object, whereas a notification will only get the raw JSON object.  This is because when sending a response to a Request, it generally requires the original Request ID.  The methods on this class automatically craft the required JSON objects for responses.

### The protocol

The protcol itself is [well documented](https://github.com/Microsoft/language-server-protocol/blob/master/protocol) by Microsoft.  It is well worth reading through all of the available messages before reading about the message router.

## Message Router

Now that we have a way to receive formatted requests and send formatted responses it's time to actually do something! This is the job of the Message Router.  It will take a request and then either respond to request itself or hand it off to a provider to provide a response.

The language protocol describes three types of messages

1. A **Request**.  This message requires a response from the other party.
   For example, the client sends a request to autocomplete items for a cursor location, and the server sends the possible autocomplete items.

2. A **Notification**.  This is a message which does not require/have a response from the other party.
   For example, the server sends a notification message to the client, to display a warning message box that the version of Puppet is too old and functionality will be limited.

3. A custom message.  This is a message that is not described in the protocol but can still be sent over the same channel.
   For example, the Puppet Language Server uses a custom request for the Node Graph Preview.  The client sends a `puppet/compileNodeGraph` request and responds with a `CompileNodeGraphResponse` object.

The [message router](https://github.com/jamespogran/puppet-vscode/blob/master/server/lib/puppet-languageserver/message_router.rb) is composed of a few modules

### Document Store

The Document Store holds copies of the documents being edited in memory, as a simple hash table.  This means documents that are being edited by the user are not saved to disk, which allows the language server to do inline linting whereas standard VS Code lint cannot.

### Crash Dump module

If the language server throws an error it cannot handle, normally it would crash.  This may seem like an odd choice, but I needed feedback when the language server didn't behave and terminating makes this very obvious.  However to help people provide us with useful feedback the crash dump module will generate a crash dump file at `%TMP%\puppet_language_server_crash.txt`.  This dump file contains all of the relevant version information, a copy of the document store, relevant backtrace, and the request that triggered the crash.

## Request Handler

The request handler (the `receive_request` method) is basically a really big case statement where the request name determines the code path.  The following messages are handled directly by the message router:

### `initialize`

When the client first connects, it sends an initialize request.  The server then responds with its capabilities.  At the time of writing these capabilities are static, however in the not too distant future, I can see the server gracefully degraded its functionality; for example, if the Puppet version is less than 4, then Puppet 4 type inspection would not be possible.  Same with providing support for structure facts, which are available in Facter 3 but not Facter2.

### `shutdown`

The shutdown method is sent to indicate the client is about to disconnect and should start its shutdown process.  The shutdown is actually handled in the TCP Server where the client disconnection takes place.

### `puppet/getVersion`

This custom request is sent by the client to get the current versions and status of the Language Server.  It returns the following version information: Puppet Version and Facter Version.  The response also sends status information; whether the facts, types and functions have been preloaded.  This information is used during the UI to show the `Loading Puppet (xx%)` message you see in the bottom right corner of VS Code.  If, for example, the functions haven't loaded then they will not be availale during hover or autocomplete requests.

The following requests are handled by providers:

### `puppet/getResource`

Similar to the [`puppet resource`](https://puppet.com/docs/puppet/latest/man/resource.html), this custom request will list all of the puppet resources for a title name.  For example a request with `typename = user` will return all of the user resources on the system.  Typically the client will only issue typename requests however the server does support adding a resource title in the request; `typename = user, title = username`.  This is handled by the PuppetHelper.

### `puppet/compileNodeGraph`

This custom request will compile the supplied manifest file and then generate a [DOT file](https://en.wikipedia.org/wiki/DOT_(graph_description_language)) which shows all of the resources, and their dependencies in the catalog.  This is handled by the PuppetParserHelper.

### `textDocument/completion`

The language client will issue a completion request when it is trying to auto-complete text, either automatically or issued via user command (Typically ctrl-space).  This request will then return with an array of short form items.  The client will then issue `completionItem/resolve` requests for detailed information.  This is needed as completion requests can generate a lot of CPU and memory overhead computing detailed information which may not be needed.  Instead the short form item format means the client can display the possible options to the user quickly, and then the user can choose to get more information as they desire.  This is handled by the CompletionProvider.

### `completionItem/resolve`

The resolution request is sent from the client when the user wishes to get more detailed information about a completion item, from a previous completion request.  This is also handled by the CompletionProvider

### `textDocument/hover`

When you hover the mouse cursor over text, the client sends hover requests to the language server.  The server can then interpret where the cursor is and provide useful information.  This is handled by the HoverProvider.

## Notification Handler

The notification handler (the `receive_notification` method), much like the request hanlder, is basically a really big case statement where the notification name determines the code path.

### `initialized`

> The initialized notification is sent from the client to the server after the client received the result of the initialize request but before the client is sending any other request or notification to the server

### `exit`

Triggers the underlying server connection to close.  For example if the underlying transport was TCP, then the server would disconnect the client.  Typically this should be sent after a `shutdown` request.

### `textDocument/didOpen` and `textDocument/didChange`

These notifications are sent as a document is opened, and as a user makes changes to the document.  At the moment, the server only supports full content change notifications, i.e. the entire file is sent in notifications.  However it is hoped to support the incremental change notifications in the near future.  Once the document is updated in the document store, a document validation is triggered and the results sent back to the client.

### `textDocument/didSave`

This is a null event for this server.

### `textDocument/didClose`

The document that is closed is removed from the document store to help save some memory.

# Wrapping up...

In this post I looked at how the JSON RPC messages are interpreted, and then actioned in the message router. In the next post we start looking at one the language providers in detail.

# Blog series links

[Part 1 - Introduction to the extension](../puppet-extension-deep-dive-part1)

[Part 2 - Introduction to the Language Server](../puppet-extension-deep-dive-part2)

[Part 3 - JSON RPC handler and message router](../puppet-extension-deep-dive-part3)

Part 4 - Language Providers
