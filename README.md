# dissonance

An async-friendly Rust library for generating Noise-encrypted transport
protocols. It provides tools for:
- creating an [`AsyncRead`] + [`AsyncWrite`] Noise socket
- creating a [`Stream`] + [`Sink`] abstraction layer on top of it (with
  separate sender and responder message types).

[`AsyncRead`]: tokio::io::AsyncRead
[`AsyncWrite`]: tokio::io::AsyncWrite
[`Stream`]: futures::stream::Stream
[`Sink`]: futures::sink::Sink

Check out https://docs.rs/dissonance/ for more information.

## Quickstart - Encrypted raw byte streams with [`NoiseSocket`]
In order to create a proper transport you first need to obtain a
[`NoiseSocket`]. This is done through the [`NoiseBuilder`] interface.

[`NoiseSocket`]: crate::noise_session::NoiseSocket
[`NoiseBuilder`]: crate::noise_session::NoiseBuilder

> [!WARNING]
> NoiseBuilder requires a reliable underlying transport protocol to work!

Suppose you want to upgrade your plain [`TcpStream`] to a Noise one as an
initiator by performing an IK Noise handshake. For this, you can use the
[`NoiseBuilder::build_as_initiator()`] method after supplying the necessary
handshake data as follows:

[`NoiseBuilder::build_as_initiator()`]:
crate::noise_session::NoiseBuilder::build_as_initiator
[`TcpStream`]: https://docs.rs/tokio/latest/tokio/net/struct.TcpStream.html

```rust
let socket = NoiseBuilder::<TcpStream>::new(my_keys, my_tcp_stream)
    .set_my_type(NoiseSelfType::I)
    .set_peer_type(NoisePeerType::K(peer_key))
    .build_as_initiator().await?;
```
The resulting `socket` provides a unified `AsyncRead + AsyncWrite` interface
for transporting raw bytes over an encrypted channel.

## Abstracting away the bytes with [`NoiseTransport`]
An [`AsyncRead`] + [`AsyncWrite`] interface is often not enough for creating
a proper communication protocol. We want to send and receive messages of known
types that aren't necessarily the same between the sender and the responder.

[`NoiseTransport`]: crate::noise_transport::NoiseTransport

As a toy example - suppose the sender is a client and the responder is
a server. Each client can request the current date from the server. The
server can then respond with a formatted date encoded in a [`String`].
Let's encode each message type by leveraging Rust's type system:
```rust
#[derive(Serialize, Deserialize)]
enum Request {
    GetDateTime
}

#[derive(Serialize, Deserialize)]
enum Response {
    DateTime(String)
}
```
By encoding each message in an `enum` we can add more requests and more
responses later. By separating request and response types from each other we
won't have to match requests when waiting for a response and vice-versa.

Both `Request` and `Response` need to be serializable and deserializable in
order to send them through a socket. This is done through Serde's
[`Serialize`] and [`Deserialize`] traits.

[`Serialize`]: serde::Serialize
[`Deserialize`]: serde::Deserialize

We can now create a [`NoiseTransport`] that abstracts away the bytes and
turns the [`AsyncRead`] + [`AsyncWrite`] Noise socket into a unified
[`Stream`] + [`Sink`] interface:

```rust
let transport = NoiseTransport::<TcpStream, Request, Response>::new(socket);
```
That was easy! We can now use the [`StreamExt`] and [`SinkExt`] traits to
send and receive our messages:
```rust
transport.send(Request::GetDateTime).await?;
let response: Response = transport.next().await?;
```

[`StreamExt`]: futures::stream::StreamExt
[`SinkExt`]: futures::sink::SinkExt

## Using the underlying socket of a [`NoiseTransport`] directly
Suppose you want to send a large file (a couple of gigabytes) to the server.
Encoding this file as a vector of bytes would not only be extremely inefficient,
it would also be impossible since either the sender or receiver could run out
of RAM. In this case the best way to send this file would be to copy it with
[`tokio::io::copy()`](https://docs.rs/tokio/latest/tokio/io/fn.copy.html)
by using the underlying socket directly.

You can get a temporary mutable reference to the underlying socket by calling
[`NoiseTransport::get_mut()`]. You can then copy the file as you normally
would by using the `copy()` method:
```rust
let mut socket = transport.get_mut();
tokio::io::copy(&mut my_file, &mut socket).await?
```
[`NoiseTransport::get_mut()`]:
crate::noise_transport::NoiseTransport::get_mut()

## Technical details
Sending messages over a stream requires serialization. This is done using
a combination of [Serde](https://docs.rs/serde/latest/serde/) with the
[Postcard](https://docs.rs/postcard/latest/postcard/) serializer.

Individual Noise messages are framed with a big endian `u16`
[`LengthDelimitedCodec`]. Then, each Noise message pack (a set of consecutive
Noise frames) gets framed with a big endian `u32` [`LengthDelimitedCodec`].
This double framing is required for sending large byte streams, as per the
previous section.

[`LengthDelimitedCodec`]: tokio_util::codec::LengthDelimitedCodec

License: BSD-3-Clause
