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
