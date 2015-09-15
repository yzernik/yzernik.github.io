---
layout: post
title:  "Connecting to the Bitcoin network with Scala and Akka"
categories: jekyll update
---
The Bitcoin network is made up of nodes that communicate over TCP using a [set of different message types][bitcoin-protocol]. Besides sending and receiving messages, a node in the [extended Bitcoin network][extended-network] can do other things at the same time, such as mining, validating transactions, running a wallet, etc.

In the reference client implementation, there is one thread that handles socket communication and one thread that processes individual peer messages. Incoming peer messages are read into a `CDataStream` buffer for each connected node, which is then processed using the [`ProcessMessage`][process-message] function. `ProcessMessage` deserializes messages as they are being processed.

In my Scala Bitcoin client implementation, I want to use case classes to represent the different message types that will be sent and received by the client. There will have to be a way to convert raw TCP data back and forth into instances of message classes that can be sent and received by the message-handling Akka actors.


## The Message Codec

I use the `scodec` library to define the codec that will [encode and decode messages][bitcoin-scodec].

In the Bitcoin protocol, there is a common message header at the beginning of each message that includes a command string that tells the type of payload that will follow. I will need to create a polymorphic codec so that I can handle messages containing all of the different payload types. This is possible using a [`Companion Type System`][companion-type], following the example given by Kifi.

I start by defining a message trait:

```scala
trait Message { self =>
  type E >: self.type <: Message
  def companion: MessageCompanion[E]
  def instance: E = self
}
```

along with it's companion trait:

```scala
trait MessageCompanion[E <: Message] {
  def codec(version: Int): Codec[E]
  def command: String
}
```

and the companion object:

```scala
object MessageCompanion {
  val all: Set[MessageCompanion[_ <: Message]] = Set(Addr, Alert, Block, GetAddr, GetBlocks,
    GetData, GetHeaders, Headers, Inv, MemPool, NotFound, Ping, Pong, Reject,
    Tx, Verack, Version)
  val byCommand: Map[ByteVector, MessageCompanion[_ <: Message]] = {
    require(all.map(_.command).size == all.size, "Type headers must be unique.")
    all.map { companion => Message.padCommand(companion.command) -> companion }.toMap
  }
}
```


Now we can use the information from the message header to implement a scodec `Codec` for the full message:

```scala
def codec(magic: Long, version: Int): Codec[Message] = {

  def encode(msg: Message) = {
    val c = msg.companion.codec(version)
    for {
      magic <- uint32L.encode(magic)
      command <- bytes(12).encode(padCommand(msg.companion.command))
      payload <- c.encode(msg)
      length <- uint32L.encode(payload.length / 8)
      chksum <- uint32L.encode(Util.checksum(payload.toByteVector))
    } yield magic ++ command ++ length ++ chksum ++ payload
  }

  def decode(bits: BitVector) =
    for {
      metadata <- decodeHeader(bits, magic, version)
      (command, length, chksum, payload, rest) = metadata
      msg <- decodePayload(payload, version, chksum, command)
    } yield msg

  Codec[Message](encode _, decode _)
}
```

The `encode` function is straightforward. It first gets the correct `Codec` for the message type that it is encoding. Then it encodes the different parts of the header and payload, and then it concatenates them all in the correct order.

The `decode` function is more complicated. It doesn't know which message type `Codec` to use until it has started decoding the message header. So we first use a `decodeHeader` function to extract the metadata from the header. Then we continue the for-comprehension (flatMap) with a second function called `decodePayload` to decode the remainder of the message.

`decodePayload` uses the information we've gathered from the header to decode the message payload:

```scala
def decodePayload(payload: BitVector, version: Int, chksum: Long, command: ByteVector) = {
  val cmd = MessageCompanion.byCommand(command)
  cmd.codec(version).decode(payload).flatMap { p =>
    if (!p.remainder.isEmpty)
      Failure(scodec.Err("payload length did not match."))
    else if (Util.checksum(payload.toByteVector) == chksum) {
      Successful(p)
    } else {
      Failure(scodec.Err("checksum did not match."))
    }
  }
}
```

and this is where the companion type becomes useful. Using just the command string from the header, we can use the `byCommand` function to get the correct message type, and therefore also the correct codec for it's payload.

Now, defining a new message type is easy and requires no changes to the `Message` trait. It only requires an implementation of the `Message` trait and an implemenation of the parametrized `MessageCompanion` trait. For example, the `GetAddr` message:

```scala
case class GetAddr() extends Message {
  type E = GetAddr
  def companion = GetAddr
}

object GetAddr extends MessageCompanion[GetAddr] {
  def codec(version: Int): Codec[GetAddr] = provide(GetAddr())
  def command = "getaddr"
}
```


## The Akka Extension

Now that I have the codec to serialize and deserialize peer messages, I can create the TCP [peer connections to other nodes][btc-io] in the Bitcoin network.

The Akka library includes a useful extension for creating TCP connections. It handles the low-level resources, and provides a simple interface for receiving and sending data. My goal is to create a second Akka extension, on top of the existing TCP extension, that will send and receive Bitcoin messages instead of raw binary data.

A real Bitcoin node should be able to both initiate and accept connections with other nodes in the network. I use Akka's `Tcp.Connect` command for outbound connections and `Tcp.Bind` command for inbound connections.

When a new outbound connection needs to be initiated, a `SocketClient` actor is spawned that attempts to open a socket connection with the remote address. If it succeeds, then it spawns a `PeerConnection` actor using the new socket.

```scala
class SocketClient(magic: Long, socketManager: ActorRef) extends Actor {
  import context.system

  def receive: Receive = {
    case connect: BTC.Connect =>
      val s = sender()
      socketManager ! Tcp.Connect(connect.remoteAddress, timeout = Some(5 seconds))
      context.become(connecting(s, connect))
  }

  def connecting(handler: ActorRef, cmd: BTC.Connect): Receive = {
    case Tcp.CommandFailed(_: Tcp.Connect) =>
      handler ! BTC.CommandFailed(cmd)
      context stop self
    case c @ Tcp.Connected(remote, local) =>
      val s = sender()
      val bc = context.actorOf(PeerConnection.props(magic, handler, s, c, false), name = "PeerConnection")
      context.watch(bc)
      bc ! PeerConnection.StartConnection
    case _: Terminated =>
      context.stop(self)
  }

}
```

A similar thing happens for accepting incoming connections. A `SocketServer` actor binds to a port and spawns a new `PeerConnection` every time that a new socket is connected on the bound port.

```scala
class SocketServer(magic: Long, socketManager: ActorRef) extends Actor {
  import context.system

  var i = 0

  def receive: Receive = {
    case bind: BTC.Bind =>
      socketManager ! Tcp.Bind(self, bind.localAddress)
      context.become(binding(bind.handler, bind))
  }

  def binding(handler: ActorRef, cmd: BTC.Bind): Receive = {
    case b @ Tcp.Bound(localAddress) =>
      handler ! BTC.Bound
      context.become(bound(handler, cmd))
    case Tcp.CommandFailed(_: Tcp.Bind) =>
      handler ! BTC.CommandFailed(cmd)
      context stop self
  }

  def bound(handler: ActorRef, cmd: BTC.Bind): Receive = {
    case c @ Tcp.Connected(remote, local) =>
      val s = sender()
      val bc = context.actorOf(PeerConnection.props(magic, handler, s, c, true), name = s"PeerConnection-$i")
      bc ! PeerConnection.StartConnection
      i += 1
  }
}
```

## The Peer Handshake

In the [peer connection handshake][peer-handshake], a node sends a Version message to the remote node. It then receives a Version message and a Verack message from the remote node. When these messages are receive, the local node sends the remote client a Verack message, and the connection is officially established.

![Features](/images/bitcoin-handshake.png "Peer Handshake")

Now, to encode this in an Akka actor, I represent the different states of the handshake as a finite state machine using the Akka `become` method:

```scala
def receive: Receive = {
  case StartConnection =>
    socketHandler ! SocketHandler.HandleSocket()
    if (!inbound)
      socketHandler ! OutgoingMessage(localV)
    context.become(awaitingVersion)
  case ConnectTimeout =>
    context.stop(self)
}

def awaitingVersion: Receive = {
  case IncomingMessage(remoteV: Version) =>
    if (inbound) {
      socketHandler ! OutgoingMessage(localV)
    }
    socketHandler ! OutgoingMessage(Verack())
    context.become(awaitingVerack(remoteV))
  case ConnectTimeout =>
    context.stop(self)
}

def awaitingVerack(remoteV: Version): Receive = {
  case IncomingMessage(verack: Verack) =>
    val verNum = reconcileVersions(localV, remoteV)
    socketHandler ! SocketHandler.SetVersion(verNum)
    handler ! BTC.Connected(remoteV, c, inbound)
    context.become(connected(remoteV))
  case ConnectTimeout =>
    context.stop(self)
}

def connected(remoteV: Version): Receive = {
  case msg: IncomingMessage =>
    stash()
  case BTC.Register(ref: ActorRef) =>
    context.watch(ref)
    connectTimer.cancel
    context.become(registered(ref, remoteV))
    unstashAll()
  case _: Terminated =>
    context.stop(self)
  case ConnectTimeout =>
    context.stop(self)
}
```

The nice thing about this representation is that the same exact receive methods are used, regardless of whether the node is making inbound or an outbound connection. All that is required is a single `inbound` parameter that modifies the behavior in two of the states.


[bitcoin-protocol]: https://en.bitcoin.it/wiki/Protocol_documentation
[extended-network]: https://github.com/aantonop/bitcoinbook/blob/develop/ch06.asciidoc#the-extended-bitcoin-network
[process-message]:  https://github.com/bitcoin/bitcoin/blob/afb0ccaf9c9e4e8fac7db3564c4e19c9218c6b03/src/main.cpp#L3848
[companion-type]:   http://eng.kifi.com/reflections-on-companion-type-systems/
[bitcoin-scodec]:   https://github.com/yzernik/bitcoin-scodec
[peer-handshake]:   https://github.com/aantonop/bitcoinbook/blob/develop/ch06.asciidoc#network_handshake
[btc-io]:           https://github.com/yzernik/btc-io
