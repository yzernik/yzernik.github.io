---
layout: post
title:  "Connecting to the Bitcoin network with Scala and Akka, Part 1"
categories: jekyll update
---
The Bitcoin network is made up of nodes that communicate over TCP using a [set of different message types][bitcoin-protocol]. Besides sending and receiving messages, a node in the [extended Bitcoin network][extended-network] can do other things at the same time, such as mining, validating transactions, running a wallet, etc.

In the reference client implementation, there is one thread that handles socket communication and one thread that processes individual peer messages. Incoming peer messages are read into a `CDataStream` buffer for each connected node, which is then processed using the [`ProcessMessage`][process-message] function. `ProcessMessage` deserializes messages as they are being processed.

In my Scala Bitcoin client implementation, I want to use case classes to represent the different message types that will be sent and received by the client. There will have to be a way to convert raw TCP data back and forth into instances of message classes that can be sent and received by the message-handling Akka actors.

That is why I created a separate libary called [bitcoin-scodec][bitcoin-scodec]

## The Message Codec

I use the `scodec` library to define the codec that will encode and decode messages.

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


[bitcoin-protocol]: https://en.bitcoin.it/wiki/Protocol_documentation
[extended-network]: https://github.com/aantonop/bitcoinbook/blob/develop/ch06.asciidoc#the-extended-bitcoin-network
[process-message]:  https://github.com/bitcoin/bitcoin/blob/afb0ccaf9c9e4e8fac7db3564c4e19c9218c6b03/src/main.cpp#L3848
[companion-type]:   http://eng.kifi.com/reflections-on-companion-type-systems/
[bitcoin-scodec]:   https://github.com/yzernik/bitcoin-scodec
