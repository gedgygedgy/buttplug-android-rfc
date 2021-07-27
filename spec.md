# Buttplug Android RFC

## Introduction

The purpose of this RFC is to establish a standardized protocol for Buttplug
communication between Android apps.

### Rationale

While we could re-use an existing transport, such as the WebSockets transport,
it has a number of disadvantages for Android apps:

* The user would need to enter a WebSocket address, which may be too confusing
  and "technical" for the average smartphone user.
* Servers must listen on a TCP port, making sure not to collide with other
  applications or listen on IP addresses other than `127.0.0.1`.
* Servers have little to no control over what applications connect to the
  socket. Permissions must be negotiated in-band, and the Buttplug protocol
  currently has no mechanism for this.
* Using the TCP/IP stack may incur unneeded extra overhead.

To address these issues, we propose a transport on Android systems that uses
Android's `Service`s and `Messenger`s for communication. This system has the
following advantages:

* No need to enter confusing WebSocket addresses. When using a Buttplug client
  app, the user simply selects a Buttplug server app from a menu (the menu may
  not even be needed if there is only a single Buttplug server app installed on
  the device.)
* The server app itself could present the user with several options for servers
  to connect to, such as connecting directly to nearby Bluetooth devices or to
  a WebSocket address.
* Because we are using Android's built-in message system, there is no chance of
  socket collision, either with TCP ports or with Unix domain sockets.
* Android has a robust permissions system which can be used to restrict which
  client apps are allowed to connect to the server app, out-of-band from the
  Buttplug protocol.
* By using Android's message-passing system instead of a TCP/IP socket, we
  eliminate whatever overhead may have been associated with using the TCP/IP
  stack.

### Challenges

Implementing such a protocol on Android has some challenges:

* Android's `Context.bindService()` method is misleading. While it presents a
  stateful stream-like interface to the user, from the server's point of view,
  the connection is stateless. `Service.onBind()` is only called once to get
  the `IBinder` that the client app should use to communicate with the
  `Service`. After the first app connects, the `Service` has no way of knowing
  if subsequent apps connect, unless they explicitly send it a message over
  the `IBinder`. Perhaps more importantly, the `Service` has no way of knowing
  when an app has disconnected through `Context.unbindService()`, as
  `Service.onUnbind()` is only called after *all* apps have disconnected.

  Because of this, we must establish a stateful interface for both the client
  app and the `Service`. After calling `Context.bindService()`, the client app
  must additionally send a message to the `Service` to establish a connection,
  as well as another message to disconnect when it is finished. (This
  complication does raise the interesting possibility of more than one
  connection within a service binding, in cases where this could be supported,
  such as a proxy server app.)
* This interface must also be fault-tolerant. Because of the datagram-like
  nature of the binding, it is hard to tell when the client is no longer
  interested in talking to the server. In cases where the client process has
  exited, this is easy: `Messenger.send()` will throw a `RemoteException`, at
  which point the connection can be considered terminated. However, in cases
  where the client process is still running but no longer interested in the
  server, we need a way for the client to communicate this.

  We should also have a ping mechanism to automatically disconnect clients that
  don't respond. The Buttplug protocol has its own in-band ping mechanism, but
  it is not enabled on all servers. It may be necessary to establish an
  out-of-band ping mechanism on the Android transport.
* When the client app connects to the server app, it may want to provide
  additional connection metadata, such as a remote WebSocket address to connect
  to. This additional metadata will need to be stored in the connect message.

## Architecture

The Android transport protocol is between two apps: the client app and the
server app. (These can be the same app, but they are logically separated.) The
server app provides two components that are of interest to the client app: the
service selector `Activity` and the `Service` itself. The server app may have
one or more selector `Activity`s and one or more `Service`s.

## Selecting a `Service`

Server apps should export an `Activity` with an `intent-filter` for intent
`io.buttplug.android.intent.action.SELECT_CLIENT_TO_SERVER_V1_SERVICE`. The
`V1` suffix represents the version of the Android transport to allow for future
improvement on this protocol, and is unrelated to the Buttplug protocol
version.

When a client app wishes to connect to a server app, it searches the
`PackageManager` for `Activity`s that match the intent filter (it can also use
`Intent.ACTION_CHOOSER` or `Intent.ACTION_PICK_ACTIVITY` for this purpose.) It
checks the permission on the `Activity` (if any), and if the client app doesn't
have the permission, it requests it from the user. It then launches the
activity.

When the activity is complete, it calls `setResult()` with an `Intent`. This
`Intent` must have extra
`io.buttplug.android.intent.extra.CLIENT_TO_SERVER_V1_SERVICE`, and may have
optional extra
`io.buttplug.android.intent.extra.CLIENT_TO_SERVER_V1_CONNECT_MESSAGE`. The
`SERVICE` extra is an explicit `Intent` that can be used to bind to the
service. The `CONNECT_MESSAGE` extra is the `Message` that should be sent to
the service when the client wishes to connect. This extra may be used to
include additional metadata in the `Message`, such as a websocket address that
was selected in the activity.

When the client app receives the `Service` information, it checks the
permission on the `Service` (if any), and if the client app doesn't have the
permission, it requests it from the user.

## Binding a `Service`

Once a `Service` has been selected, the client can bind to it using
`Context.bindService()`. The `Intent` that the client provides contains the
`Service`'s package name and class name, as is usual for explicit `Intent`s. It
must also have an action of
`io.buttplug.android.intent.action.CLIENT_TO_SERVER_V1`. The `V1` suffix
represents the version of the Android transport to allow for future improvement
on this protocol, and is unrelated to the Buttplug protocol version. The
`Intent` must not have any extra data associated with it.

The `Service` then returns to the client app an `IBinder` constructed from a
`Messenger`. This `Messenger` is the binding `Messenger`, and listens for
connection requests from the client app.

## `Service` Constants

We define the following integer constants for use in interacting with a
`Service` that implements the transport protocol:

* `WHAT_ERROR` = `0`
* `WHAT_CONNECT` = `1`
* `WHAT_CONNECT_ACK` = `2`
* `WHAT_DISCONNECT` = `3`
* `WHAT_DISCONNECT_ACK` = `4`
* `WHAT_STRING_JSON_MESSAGE` = `5`
* `ERROR_BUSY` = `1`
* `ERROR_INVALID_MESSAGE` = `2`

## Connecting to a Server

Once the `Service` has been bound, the client app creates a `Messenger` from
the returned `IBinder`. When the client app wishes to connect to a server
provided by the `Service`, it creates a `Handler` and a `Messenger` that passes
to this `Handler`. This `Messenger` is the client `Messenger`. The client then
sends a `Message` through the binding `Messenger`. This `Message` has a `what`
value of `WHAT_CONNECT` and has the client `Messenger` in its `replyTo` field.
The `Message` may also contain `Service`-specific data in `arg1`, `arg2`,
`obj`, and `getData()` (for example, a WebSocket address to connect to a remote
server.) This specific data should be returned by the selector activity. Once
the `WHAT_CONNECT` message has been sent, the new connection is in the
"connecting" state. No messages may yet be sent in either direction, except
that the `Service` may send a `WHAT_CONNECT_ACK` or `WHAT_ERROR` message to the
client `Messenger`.

When the `Service` receives the connection request message on the binding
`Messenger`, it may either acknowledge the request and establish a connection
or reject it. If the `Service` acknowledges the request, it creates a `Handler`
and a `Messenger` that passes to this `Handler`. This `Messenger` is the server
`Messenger`. The server then sends a `Message` to the client `Messenger`. This
`Message` has a `what` value of `WHAT_CONNECT_ACK` and has the server
`Messenger` in its `replyTo` field. At this point, the connection is in the
"connected" state, and the client and server may begin exchanging Buttplug
messages.

If the `Service` rejects the connection request, it sends a `Message` to the
client `Messenger`. This `Message` has a `what` value of `WHAT_ERROR`, and an
`arg1` value equal to one of the `ERROR_*` constants describing the error. At
this point, the connection is in the "disconnected" state, and no further
messages may be sent in either direction.

Once the client has established a connection, it must not call
`Context.unbindService()` on the `Service` until it has closed all of its
connections. When `Context.unbindService()` is called, there is no guarantee
that the `Service` will continue running.

## Exchanging Messages

Once the client and server have established a connection, they can begin
exchanging Buttplug messages. Either the client or the server can send a
message at any point. When the server wishes to send a message, it sends it
to the client `Messenger`. When the client wishes to send a message, it sends
it to the server `Messenger`. In both cases, the message must be a string JSON
message. The `Message` has a `what` value of `WHAT_STRING_JSON_MESSAGE`, and
its `getData()` has a `CharSequence` field named `content`, which contains the
JSON string sent by the client or server.

## Closing a Connection

Either the client or the server may gracefully close the connection at any
point. If the client wishes to close the connection, it sends a message to the
server `Messenger`. If the server messenger wishes to close the connection, it
sends a message to the client `Messenger`. In both cases, the message has a
`what` value of `WHAT_DISCONNECT`. At this point, the connection is in the
"disconnecting" state. The `Messenger` that received the `WHAT_DISCONNECT` may
not have any further messages sent to it, and the `Messenger` of the peer that
sent it may not have any further messages sent to it except a
`WHAT_DISCONNECT_ACK` message.

When the peer receives a `WHAT_DISCONNECT` message, it sends back a
`WHAT_DISCONNECT_ACK` message. At this point, the connection is in the
"disconnected" state, and no further messages may be sent in either direction.

If at any point `Messenger.send()` throws a `RemoteException` on either end,
the connection immediately moves to the "disconnected" state, and no further
messages may be sent in either direction.
